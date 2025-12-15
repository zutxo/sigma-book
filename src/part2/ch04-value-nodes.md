# Chapter 4: Value Nodes

## Prerequisites

- AST and tree traversal concepts
- Prior chapters: [Chapter 2](../part1/ch02-type-system.md), [Chapter 3](../part1/ch03-ergotree-structure.md)

## Learning Objectives

- Understand the `Value` base type and its role in ErgoTree
- Identify different constant value types
- Explain how evaluation works via `eval`
- Work with compound values (collections, tuples)

## The Value Base Type

All ErgoTree expression nodes are values with type, opcode, and cost[^1][^2]:

```zig
/// Base type for all ErgoTree expression nodes
const Value = struct {
    tpe: SType,
    op_code: OpCode,

    /// Evaluate this node in the given environment
    pub fn eval(self: *const Value, env: *const DataEnv, evaluator: *Evaluator) !Any {
        // Default: must be overridden
        return error.NotImplemented;
    }

    /// Add fixed cost to accumulator
    pub fn addCost(self: *const Value, evaluator: *Evaluator, cost: FixedCost) void {
        evaluator.addCost(cost, self.op_code);
    }

    /// Add per-item cost for known iteration count
    pub fn addSeqCost(
        self: *const Value,
        evaluator: *Evaluator,
        cost: PerItemCost,
        n_items: usize,
    ) void {
        evaluator.addSeqCost(cost, n_items, self.op_code);
    }
};
```

## Value Hierarchy

```
Value
├── Constant
│   ├── BooleanConstant (TrueLeaf, FalseLeaf)
│   ├── ByteConstant, ShortConstant, IntConstant, LongConstant
│   ├── BigIntConstant, UnsignedBigIntConstant (v6+)
│   ├── GroupElementConstant
│   ├── SigmaPropConstant
│   ├── CollectionConstant
│   └── UnitConstant
├── ConstantPlaceholder
├── Tuple
├── ConcreteCollection
├── SigmaPropValue
│   ├── BoolToSigmaProp
│   ├── CreateProveDlog
│   ├── CreateProveDHTuple
│   ├── SigmaAnd
│   └── SigmaOr
└── Transformer (collection operations)
    ├── AND, OR, XorOf
    ├── Map, Filter, Fold
    └── Exists, ForAll
```

## Constant Values

Constants are pre-evaluated values embedded in the tree[^3][^4]:

```zig
const Constant = struct {
    tpe: SType,
    value: Literal,

    pub const COST = FixedCost{ .value = 5 }; // JitCost units

    pub fn eval(self: *const Constant, env: *const DataEnv, E: *Evaluator) Any {
        E.addCost(COST, OpCode.Constant);
        return self.value.toAny();
    }
};

/// Literal values for constants
const Literal = union(enum) {
    boolean: bool,
    byte: i8,
    short: i16,
    int: i32,
    long: i64,
    big_int: BigInt256,
    unsigned_big_int: UnsignedBigInt256,
    group_element: EcPoint,
    sigma_prop: SigmaProp,
    coll: Collection,
    tuple: []const Literal,
    unit: void,

    pub fn toAny(self: Literal) Any {
        return switch (self) {
            .boolean => |b| .{ .boolean = b },
            .int => |i| .{ .int = i },
            // ... other cases
        };
    }
};
```

### Primitive Constant Factories

```zig
pub fn intConstant(value: i32) Constant {
    return .{
        .tpe = SType.int,
        .value = .{ .int = value },
    };
}

pub fn longConstant(value: i64) Constant {
    return .{
        .tpe = SType.long,
        .value = .{ .long = value },
    };
}

pub fn byteArrayConstant(bytes: []const u8) Constant {
    return .{
        .tpe = .{ .coll = &SType.byte },
        .value = .{ .coll = .{ .bytes = bytes } },
    };
}
```

### Boolean Singletons

Boolean has special singleton instances for efficiency[^5]:

```zig
pub const TrueLeaf = Constant{
    .tpe = SType.boolean,
    .value = .{ .boolean = true },
};

pub const FalseLeaf = Constant{
    .tpe = SType.boolean,
    .value = .{ .boolean = false },
};

pub fn booleanConstant(v: bool) *const Constant {
    return if (v) &TrueLeaf else &FalseLeaf;
}
```

### Cryptographic Constants

```zig
pub fn groupElementConstant(point: EcPoint) Constant {
    return .{
        .tpe = SType.group_element,
        .value = .{ .group_element = point },
    };
}

pub fn sigmaPropConstant(prop: SigmaProp) Constant {
    return .{
        .tpe = SType.sigma_prop,
        .value = .{ .sigma_prop = prop },
    };
}

/// Group generator - base point G of secp256k1
pub const GroupGenerator = struct {
    pub const COST = FixedCost{ .value = 10 };

    pub fn eval(_: *const @This(), _: *const DataEnv, E: *Evaluator) GroupElement {
        E.addCost(COST, OpCode.GroupGenerator);
        return crypto.SECP256K1_GENERATOR;
    }
};
```

## Constant Placeholders

When constant segregation is enabled, placeholders reference the constants array[^6][^7]:

```zig
const ConstantPlaceholder = struct {
    index: u32,
    tpe: SType,

    pub const COST = FixedCost{ .value = 1 }; // Cheaper than Constant

    pub fn eval(self: *const ConstantPlaceholder, _: *const DataEnv, E: *Evaluator) !Any {
        const c = E.constants[self.index];
        E.addCost(COST, OpCode.ConstantPlaceholder);

        // Type check
        if (c.tpe != self.tpe) {
            return error.TypeMismatch;
        }
        return c.value.toAny();
    }
};
```

## Collection Values

### CollectionConstant

For collections of constant values:

```zig
const CollectionConstant = struct {
    elem_type: SType,
    items: union(enum) {
        bytes: []const u8,
        ints: []const i32,
        longs: []const i64,
        bools: []const bool,
        any: []const Literal,
    },

    pub fn tpe(self: *const CollectionConstant) SType {
        return .{ .coll = &self.elem_type };
    }
};
```

### ConcreteCollection

For collections of non-constant expressions[^8]:

```zig
const ConcreteCollection = struct {
    items: []const *Value,
    elem_type: SType,

    pub const COST = FixedCost{ .value = 20 };

    pub fn eval(self: *const ConcreteCollection, env: *const DataEnv, E: *Evaluator) ![]Any {
        E.addCost(COST, OpCode.ConcreteCollection);

        var result = try E.allocator.alloc(Any, self.items.len);
        for (self.items, 0..) |item, i| {
            result[i] = try item.eval(env, E);
        }
        return result;
    }
};
```

## Tuple Values

Heterogeneous fixed-size sequences[^9]:

```zig
const Tuple = struct {
    items: []const *Value,

    pub const COST = FixedCost{ .value = 15 };

    pub fn tpe(self: *const Tuple) STuple {
        var types = try allocator.alloc(SType, self.items.len);
        for (self.items, 0..) |item, i| {
            types[i] = item.tpe;
        }
        return STuple{ .items = types };
    }

    pub fn eval(self: *const Tuple, env: *const DataEnv, E: *Evaluator) !TupleValue {
        // Note: v5.0 only supports pairs (2 elements)
        if (self.items.len != 2) {
            return error.InvalidTupleSize;
        }

        const x = try self.items[0].eval(env, E);
        const y = try self.items[1].eval(env, E);
        E.addCost(COST, OpCode.Tuple);

        return .{ x, y };
    }
};
```

## Sigma Proposition Values

### BoolToSigmaProp

Converts boolean to cryptographic proposition[^10]:

```zig
const BoolToSigmaProp = struct {
    input: *Value, // Must be boolean

    pub const COST = FixedCost{ .value = 15 };

    pub fn eval(self: *const BoolToSigmaProp, env: *const DataEnv, E: *Evaluator) !SigmaProp {
        const v = try self.input.eval(env, E);
        E.addCost(COST, OpCode.BoolToSigmaProp);

        return SigmaProp.fromBool(v.boolean);
    }
};
```

### CreateProveDlog

Creates discrete log proposition (standard public key)[^11]:

```zig
const CreateProveDlog = struct {
    input: *Value, // GroupElement

    pub const COST = FixedCost{ .value = 10 };

    pub fn eval(self: *const CreateProveDlog, env: *const DataEnv, E: *Evaluator) !SigmaProp {
        const point = try self.input.eval(env, E);
        E.addCost(COST, OpCode.ProveDlog);

        return SigmaProp{
            .prove_dlog = ProveDlog{ .h = point.group_element },
        };
    }
};
```

### CreateProveDHTuple

Creates Diffie-Hellman tuple proposition:

```zig
const CreateProveDHTuple = struct {
    g: *Value,
    h: *Value,
    u: *Value,
    v: *Value,

    pub const COST = FixedCost{ .value = 20 };

    pub fn eval(self: *const CreateProveDHTuple, env: *const DataEnv, E: *Evaluator) !SigmaProp {
        const g_val = try self.g.eval(env, E);
        const h_val = try self.h.eval(env, E);
        const u_val = try self.u.eval(env, E);
        const v_val = try self.v.eval(env, E);
        E.addCost(COST, OpCode.ProveDHTuple);

        return SigmaProp{
            .prove_dh_tuple = ProveDhTuple{
                .g = g_val.group_element,
                .h = h_val.group_element,
                .u = u_val.group_element,
                .v = v_val.group_element,
            },
        };
    }
};
```

### SigmaAnd / SigmaOr

Combine sigma propositions[^12]:

```zig
const SigmaAnd = struct {
    items: []const *Value, // SigmaPropValues

    pub const COST = PerItemCost{
        .base = 10,
        .per_chunk = 2,
        .chunk_size = 1,
    };

    pub fn eval(self: *const SigmaAnd, env: *const DataEnv, E: *Evaluator) !SigmaProp {
        var props = try E.allocator.alloc(SigmaProp, self.items.len);
        for (self.items, 0..) |item, i| {
            props[i] = (try item.eval(env, E)).sigma_prop;
        }
        E.addSeqCost(COST, self.items.len, OpCode.SigmaAnd);

        return SigmaProp{ .cand = Cand{ .children = props } };
    }
};

const SigmaOr = struct {
    items: []const *Value,

    pub const COST = PerItemCost{
        .base = 10,
        .per_chunk = 2,
        .chunk_size = 1,
    };

    pub fn eval(self: *const SigmaOr, env: *const DataEnv, E: *Evaluator) !SigmaProp {
        var props = try E.allocator.alloc(SigmaProp, self.items.len);
        for (self.items, 0..) |item, i| {
            props[i] = (try item.eval(env, E)).sigma_prop;
        }
        E.addSeqCost(COST, self.items.len, OpCode.SigmaOr);

        return SigmaProp{ .cor = Cor{ .children = props } };
    }
};
```

## Logical Operations

### AND / OR with Short-Circuit

Boolean operations support short-circuit evaluation[^13]:

```zig
const AND = struct {
    input: *Value, // Collection[Boolean]

    pub const COST = PerItemCost{
        .base = 10,
        .per_chunk = 5,
        .chunk_size = 32,
    };

    pub fn eval(self: *const AND, env: *const DataEnv, E: *Evaluator) !bool {
        const coll = try self.input.eval(env, E);
        const items = coll.coll.bools;

        var result = true;
        var i: usize = 0;

        // Short-circuit: stop on first false
        while (i < items.len and result) {
            result = result and items[i];
            i += 1;
        }

        // Cost based on actual items processed
        E.addSeqCost(COST, i, OpCode.And);
        return result;
    }
};

const OR = struct {
    input: *Value, // Collection[Boolean]

    pub const COST = PerItemCost{
        .base = 10,
        .per_chunk = 5,
        .chunk_size = 32,
    };

    pub fn eval(self: *const OR, env: *const DataEnv, E: *Evaluator) !bool {
        const coll = try self.input.eval(env, E);
        const items = coll.coll.bools;

        var result = false;
        var i: usize = 0;

        // Short-circuit: stop on first true
        while (i < items.len and !result) {
            result = result or items[i];
            i += 1;
        }

        E.addSeqCost(COST, i, OpCode.Or);
        return result;
    }
};
```

### XorOf

XOR over boolean collection:

```zig
const XorOf = struct {
    input: *Value,

    pub const COST = PerItemCost{
        .base = 20,
        .per_chunk = 5,
        .chunk_size = 32,
    };

    pub fn eval(self: *const XorOf, env: *const DataEnv, E: *Evaluator) !bool {
        const coll = try self.input.eval(env, E);
        const items = coll.coll.bools;

        var result = false;
        for (items) |b| {
            result = result != b; // XOR
        }

        E.addSeqCost(COST, items.len, OpCode.XorOf);
        return result;
    }
};
```

## Cost Summary

| Operation | Cost Type | Value |
|-----------|-----------|-------|
| Constant | Fixed | 5 |
| ConstantPlaceholder | Fixed | 1 |
| Tuple | Fixed | 15 |
| BoolToSigmaProp | Fixed | 15 |
| CreateProveDlog | Fixed | 10 |
| CreateProveDHTuple | Fixed | 20 |
| GroupGenerator | Fixed | 10 |
| AND/OR | PerItem | base=10, chunk=5/32 |
| SigmaAnd/SigmaOr | PerItem | base=10, chunk=2/1 |
| XorOf | PerItem | base=20, chunk=5/32 |

## Summary

- `Value` is the base for all ErgoTree expression nodes
- Every value has `tpe` (type), `op_code`, and cost info
- Constants are pre-evaluated; `ConstantPlaceholder` references segregated constants
- Sigma propositions produce cryptographic statements for proving
- Boolean ops support short-circuit evaluation with accurate cost tracking
- The `eval` method on each value defines its evaluation semantics

---

*Next: [Chapter 5: Operations and Opcodes](./ch05-operations-opcodes.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala:30-165`

[^2]: Rust: `ergotree-ir/src/mir/expr.rs:1-80`

[^3]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala:305-398`

[^4]: Rust: `ergotree-ir/src/mir/constant.rs:51-58`

[^5]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala` (TrueLeaf, FalseLeaf)

[^6]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala:400-422`

[^7]: Rust: `ergotree-ir/src/mir/constant/constant_placeholder.rs`

[^8]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala` (ConcreteCollection)

[^9]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala:771-810`

[^10]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala:28-57`

[^11]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala` (CreateProveDlog)

[^12]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala` (SigmaAnd, SigmaOr)

[^13]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala:186-299`
