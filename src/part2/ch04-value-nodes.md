# Chapter 4: Value Nodes

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Understanding of Abstract Syntax Trees (ASTs) as hierarchical representations where each node represents a language construct
- Tree traversal techniques (depth-first evaluation)
- Prior chapters: [Chapter 2](../part1/ch02-type-system.md) for the type system that governs value types, [Chapter 3](../part1/ch03-ergotree-structure.md) for how values are serialized in ErgoTree

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the `Value` base type and its role as the foundation for all ErgoTree expression nodes
- Distinguish between different constant value types (primitives, cryptographic, collections)
- Describe how the `eval` method implements the evaluation semantics for each node type
- Work with compound values including collections and tuples

## The Value Base Type

ErgoTree is fundamentally an expression tree where every node produces a typed value. The `Value` base type defines the common interface that all expression nodes share—a type annotation, an opcode for serialization, and an evaluation method that computes the result[^1][^2].

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

The hierarchy divides into several major categories:
- **Constants** hold literal values known at compile time
- **ConstantPlaceholder** references segregated constants by index (see Chapter 3)
- **Compound values** (Tuple, ConcreteCollection) combine multiple values
- **SigmaPropValue** nodes produce cryptographic propositions for signing
- **Transformers** perform operations on collections

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

When constant segregation is enabled (Chapter 3), placeholders replace inline constants with index references into the constants array[^6][^7]. This separation enables template caching—the same expression tree structure can be reused with different constant values. Placeholder evaluation costs less than inline constants because the constant data has already been parsed and validated during ErgoTree deserialization.

```zig
const ConstantPlaceholder = struct {
    index: u32,
    tpe: SType,

    pub const COST = FixedCost{ .value = 1 }; // Cheaper than Constant

    pub fn eval(self: *const ConstantPlaceholder, _: *const DataEnv, E: *Evaluator) !Any {
        // Bounds check first (prevents out-of-bounds access)
        if (self.index >= E.constants.len) {
            return error.ConstantIndexOutOfBounds;
        }
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

ErgoTree supports two kinds of collection nodes, optimized for different use cases:

### CollectionConstant

For collections where all elements are known at compile time, `CollectionConstant` stores the values directly. This enables efficient serialization and avoids evaluation overhead for static data like byte arrays and fixed integer sequences.

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

When collection elements are computed expressions rather than literals, `ConcreteCollection` holds references to sub-expression nodes[^8]. Each element is evaluated at runtime, making this suitable for dynamically constructed collections.

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
// NOTE: In production, use a pre-allocated value pool to avoid dynamic
// allocation during evaluation. See ZIGMA_STYLE.md memory management section.
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

This chapter introduced the value node hierarchy that forms the foundation of ErgoTree's expression tree:

- **`Value`** is the base type for all ErgoTree expression nodes, defining the common interface of type, opcode, and evaluation method
- Every value carries type information (`tpe`) used for static type checking and cost information used for bounded execution
- **Constants** are pre-evaluated literals embedded in the tree; **`ConstantPlaceholder`** provides indirection to segregated constants for template sharing
- **Collection values** come in two forms: `CollectionConstant` for static data and `ConcreteCollection` for computed elements
- **Sigma proposition values** (`CreateProveDlog`, `CreateProveDHTuple`, `SigmaAnd`, `SigmaOr`) produce cryptographic propositions that require zero-knowledge proofs
- **Boolean operations** (`AND`, `OR`) support short-circuit evaluation, charging costs only for elements actually processed
- The `eval` method on each value type implements its evaluation semantics, transforming the AST node into a runtime value

---

*Next: [Chapter 5: Operations and Opcodes](./ch05-operations-opcodes.md)*

[^1]: Scala: [`values.scala:30-165`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/values.scala#L30-L165)

[^2]: Rust: [`expr.rs:1-80`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/expr.rs#L1-L80)

[^3]: Scala: [`values.scala:305-398`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/values.scala#L305-L398)

[^4]: Rust: [`constant.rs:51-58`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/constant.rs#L51-L58)

[^5]: Scala: [`values.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/values.scala) (TrueLeaf, FalseLeaf)

[^6]: Scala: [`values.scala:400-422`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/values.scala#L400-L422)

[^7]: Rust: [`constant_placeholder.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/constant/constant_placeholder.rs)

[^8]: Scala: [`values.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/values.scala) (ConcreteCollection)

[^9]: Scala: [`values.scala:771-810`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/values.scala#L771-L810)

[^10]: Scala: [`trees.scala:28-57`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/trees.scala#L28-L57)

[^11]: Scala: [`trees.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/trees.scala) (CreateProveDlog)

[^12]: Scala: [`trees.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/trees.scala) (SigmaAnd, SigmaOr)

[^13]: Scala: [`trees.scala:186-299`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/trees.scala#L186-L299)