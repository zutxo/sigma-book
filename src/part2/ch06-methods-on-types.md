# Chapter 6: Methods on Types

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Understanding of method dispatch—how method calls are resolved to specific implementations based on the receiver type
- Familiarity with type hierarchies and how types can share common method interfaces
- Prior chapters: [Chapter 2](../part1/ch02-type-system.md) for type codes used in method resolution, [Chapter 5](./ch05-operations-opcodes.md) for operations vs methods distinction

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain how methods are organized via `MethodsContainer` and resolved by type code and method ID
- Use methods on numeric, collection, box, and cryptographic types
- Describe the method resolution process from `MethodCall` to method implementation
- Access transaction context and blockchain state through context methods

## Method Architecture

While Chapter 5 covered standalone operations (arithmetic, comparisons, etc.), ErgoTree also supports methods—operations that belong to specific types. The distinction matters for serialization: operations use opcodes directly, while method calls serialize a type code, method ID, and arguments. This design allows types to have rich APIs without consuming the limited opcode space.

Methods are organized through a `MethodsContainer` system that groups related methods by their receiver type[^1][^2]:

```
Method Organization
══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│                    STypeCompanion                              │
│    (type_code: u8, methods: []const SMethodDesc)               │
└───────────────────────┬────────────────────────────────────────┘
                        │
       ┌────────────────┼────────────────┬───────────────────┐
       ▼                ▼                ▼                   ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ SNumeric     │ │ SBox         │ │ SColl        │ │ SContext     │
│ TYPE_CODE=2-6│ │ TYPE_CODE=99 │ │ TYPE_CODE=12 │ │ TYPE_CODE=101│
├──────────────┤ ├──────────────┤ ├──────────────┤ ├──────────────┤
│ toByte   (1) │ │ value    (1) │ │ size     (1) │ │ dataInputs(1)│
│ toShort  (2) │ │ propBytes(2) │ │ getOrElse(2) │ │ headers  (2) │
│ toInt    (3) │ │ bytes    (3) │ │ map      (3) │ │ preHeader(3) │
│ toLong   (4) │ │ id       (5) │ │ exists   (4) │ │ INPUTS   (4) │
│ toBigInt (5) │ │ getReg   (7) │ │ forall   (5) │ │ OUTPUTS  (5) │
│ toBytes  (6) │ │ tokens   (8) │ │ fold     (6) │ │ HEIGHT   (6) │
│ ...          │ │ ...          │ │ ...          │ │ ...          │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

Method Resolution:
  MethodCall(receiver, type_code=99, method_id=1)
       │
       ▼
  resolveMethod(99, 1) → SBoxMethods.VALUE
       │
       ▼
  method.eval(receiver, args, evaluator)
```

```zig
const MethodsContainer = struct {
    type_code: u8,
    methods: []const SMethod,

    pub fn getMethodById(self: *const MethodsContainer, method_id: u8) ?*const SMethod {
        for (self.methods) |*m| {
            if (m.method_id == method_id) return m;
        }
        return null;
    }

    pub fn getMethodByName(self: *const MethodsContainer, name: []const u8) ?*const SMethod {
        for (self.methods) |*m| {
            if (std.mem.eql(u8, m.name, name)) return m;
        }
        return null;
    }
};

const SMethod = struct {
    obj_type: STypeCompanion,
    name: []const u8,
    method_id: u8,
    tpe: SFunc,
    cost_kind: CostKind,
    min_version: ?ErgoTreeVersion = null, // v6+ methods

    pub fn eval(
        self: *const SMethod,
        receiver: Any,
        args: []const Any,
        E: *Evaluator,
    ) !Any {
        // Method dispatch by type_code and method_id
        return try evalMethod(
            self.obj_type.type_code,
            self.method_id,
            receiver,
            args,
            E,
        );
    }
};
```

## Available Method Containers

| Type | Container | Method Count |
|------|-----------|--------------|
| Byte, Short, Int, Long | SNumericMethods | 13 |
| BigInt | SBigIntMethods | 13 |
| UnsignedBigInt (v6+) | SUnsignedBigIntMethods | 13 |
| Boolean | SBooleanMethods | 0 |
| GroupElement | SGroupElementMethods | 4 |
| SigmaProp | SSigmaPropMethods | 2 |
| Box | SBoxMethods | 10 |
| Coll[T] | SCollectionMethods | 20 |
| Option[T] | SOptionMethods | 4 |
| Context | SContextMethods | 12 |
| Header | SHeaderMethods | 16 |
| PreHeader | SPreHeaderMethods | 8 |
| AvlTree | SAvlTreeMethods | 9 |
| Global | SGlobalMethods | 4 |

## Numeric Type Methods

All numeric types share common methods[^3][^4]:

```zig
const SNumericMethods = struct {
    pub const TYPE_CODE = 0; // Varies by actual type

    // Conversion methods (v5+)
    pub const TO_BYTE = SMethod{
        .method_id = 1,
        .name = "toByte",
        .tpe = SFunc.unary(.{ .type_var = "T" }, .byte),
        .cost_kind = .{ .type_based = .{ .primitive = 5, .big_int = 10 } },
    };

    pub const TO_SHORT = SMethod{ .method_id = 2, .name = "toShort", ... };
    pub const TO_INT = SMethod{ .method_id = 3, .name = "toInt", ... };
    pub const TO_LONG = SMethod{ .method_id = 4, .name = "toLong", ... };
    pub const TO_BIGINT = SMethod{ .method_id = 5, .name = "toBigInt", ... };

    // Binary representation (v6+)
    pub const TO_BYTES = SMethod{
        .method_id = 6,
        .name = "toBytes",
        .tpe = SFunc.unary(.{ .type_var = "T" }, .{ .coll = &SType.byte }),
        .cost_kind = .{ .fixed = 5 },
        .min_version = .v3, // v6
    };

    pub const TO_BITS = SMethod{
        .method_id = 7,
        .name = "toBits",
        .tpe = SFunc.unary(.{ .type_var = "T" }, .{ .coll = &SType.boolean }),
        .cost_kind = .{ .fixed = 5 },
        .min_version = .v3,
    };

    // Bitwise operations (v6+)
    pub const BITWISE_INVERSE = SMethod{ .method_id = 8, .name = "bitwiseInverse", ... };
    pub const BITWISE_OR = SMethod{ .method_id = 9, .name = "bitwiseOr", ... };
    pub const BITWISE_AND = SMethod{ .method_id = 10, .name = "bitwiseAnd", ... };
    pub const BITWISE_XOR = SMethod{ .method_id = 11, .name = "bitwiseXor", ... };
    pub const SHIFT_LEFT = SMethod{ .method_id = 12, .name = "shiftLeft", ... };
    pub const SHIFT_RIGHT = SMethod{ .method_id = 13, .name = "shiftRight", ... };
};
```

### Numeric Method Summary

| ID | Method | Signature | v5 | v6 | Description |
|----|--------|-----------|----|----|-------------|
| 1 | toByte | T => Byte | ✓ | ✓ | Convert (may overflow) |
| 2 | toShort | T => Short | ✓ | ✓ | Convert (may overflow) |
| 3 | toInt | T => Int | ✓ | ✓ | Convert (may overflow) |
| 4 | toLong | T => Long | ✓ | ✓ | Convert (may overflow) |
| 5 | toBigInt | T => BigInt | ✓ | ✓ | Convert (always safe) |
| 6 | toBytes | T => Coll[Byte] | - | ✓ | Big-endian bytes |
| 7 | toBits | T => Coll[Bool] | - | ✓ | Bit representation |
| 8 | bitwiseInverse | T => T | - | ✓ | Bitwise NOT |
| 9 | bitwiseOr | (T,T) => T | - | ✓ | Bitwise OR |
| 10 | bitwiseAnd | (T,T) => T | - | ✓ | Bitwise AND |
| 11 | bitwiseXor | (T,T) => T | - | ✓ | Bitwise XOR |
| 12 | shiftLeft | (T,Int) => T | - | ✓ | Left shift |
| 13 | shiftRight | (T,Int) => T | - | ✓ | Arithmetic right shift |

## Collection Methods

Collections have the richest method set[^5][^6]:

```zig
const SCollectionMethods = struct {
    pub const TYPE_CODE = 12;

    // Basic access
    pub const SIZE = SMethod{
        .method_id = 1,
        .name = "size",
        .tpe = SFunc.unary(.{ .coll = .{ .type_var = "T" } }, .int),
        .cost_kind = .{ .fixed = 14 },
    };

    pub const GET_OR_ELSE = SMethod{
        .method_id = 2,
        .name = "getOrElse",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "T" } },
            .int,
            .{ .type_var = "T" },
        }, .{ .type_var = "T" }),
        .cost_kind = .dynamic,
    };

    // Transformation
    pub const MAP = SMethod{
        .method_id = 3,
        .name = "map",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "IV" } },
            SFunc.unary(.{ .type_var = "IV" }, .{ .type_var = "OV" }),
        }, .{ .coll = .{ .type_var = "OV" } }),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 1, .chunk_size = 10 } },
    };

    pub const FILTER = SMethod{
        .method_id = 8,
        .name = "filter",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "IV" } },
            SFunc.unary(.{ .type_var = "IV" }, .boolean),
        }, .{ .coll = .{ .type_var = "IV" } }),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 1, .chunk_size = 10 } },
    };

    pub const FOLD = SMethod{
        .method_id = 6,
        .name = "fold",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "IV" } },
            .{ .type_var = "OV" },
            SFunc.binary(.{ .type_var = "OV" }, .{ .type_var = "IV" }, .{ .type_var = "OV" }),
        }, .{ .type_var = "OV" }),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 1, .chunk_size = 10 } },
    };

    // Predicates
    pub const EXISTS = SMethod{
        .method_id = 4,
        .name = "exists",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "IV" } },
            SFunc.unary(.{ .type_var = "IV" }, .boolean),
        }, .boolean),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 1, .chunk_size = 10 } },
    };

    pub const FORALL = SMethod{
        .method_id = 5,
        .name = "forall",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "IV" } },
            SFunc.unary(.{ .type_var = "IV" }, .boolean),
        }, .boolean),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 1, .chunk_size = 10 } },
    };

    // Combination
    pub const APPEND = SMethod{
        .method_id = 9,
        .name = "append",
        .tpe = SFunc.binary(
            .{ .coll = .{ .type_var = "IV" } },
            .{ .coll = .{ .type_var = "IV" } },
            .{ .coll = .{ .type_var = "IV" } },
        ),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 2, .chunk_size = 100 } },
    };

    pub const SLICE = SMethod{
        .method_id = 10,
        .name = "slice",
        .tpe = SFunc.new(&[_]SType{
            .{ .coll = .{ .type_var = "IV" } },
            .int,
            .int,
        }, .{ .coll = .{ .type_var = "IV" } }),
        .cost_kind = .{ .per_item = .{ .base = 10, .per_chunk = 2, .chunk_size = 100 } },
    };

    pub const ZIP = SMethod{
        .method_id = 14,
        .name = "zip",
        .cost_kind = .{ .per_item = .{ .base = 10, .per_chunk = 1, .chunk_size = 10 } },
    };

    // Index operations
    pub const INDICES = SMethod{
        .method_id = 11,
        .name = "indices",
        .tpe = SFunc.unary(.{ .coll = .{ .type_var = "T" } }, .{ .coll = &SType.int }),
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 2, .chunk_size = 128 } },
    };

    pub const INDEX_OF = SMethod{
        .method_id = 26,
        .name = "indexOf",
        .cost_kind = .{ .per_item = .{ .base = 20, .per_chunk = 1, .chunk_size = 10 } },
    };
};
```

### Collection Method Summary

| ID | Method | v5 | v6 | Description |
|----|--------|----|----|-------------|
| 1 | size | ✓ | ✓ | Number of elements |
| 2 | getOrElse | ✓ | ✓ | Element with default |
| 3 | map | ✓ | ✓ | Transform elements |
| 4 | exists | ✓ | ✓ | Any match predicate |
| 5 | forall | ✓ | ✓ | All match predicate |
| 6 | fold | ✓ | ✓ | Reduce to single value |
| 7 | apply | ✓ | ✓ | Element at index (panics if OOB) |
| 8 | filter | ✓ | ✓ | Keep matching elements |
| 9 | append | ✓ | ✓ | Concatenate |
| 10 | slice | ✓ | ✓ | Extract range |
| 14 | indices | ✓ | ✓ | Range 0..size-1 |
| 15 | flatMap | ✓ | ✓ | Map and flatten |
| 19 | patch | ✓ | ✓ | Replace range |
| 20 | updated | ✓ | ✓ | Replace at index |
| 21 | updateMany | ✓ | ✓ | Batch update |
| 26 | indexOf | ✓ | ✓ | Find element index |
| 29 | zip | ✓ | ✓ | Pair with other collection |
| 30 | reverse | - | ✓ | Reverse order |
| 31 | startsWith | - | ✓ | Prefix match |
| 32 | endsWith | - | ✓ | Suffix match |
| 33 | get | - | ✓ | Safe element access (returns Option) |

## Box Methods

Access box properties[^7][^8]:

```zig
const SBoxMethods = struct {
    pub const TYPE_CODE = 99;

    pub const VALUE = SMethod{
        .method_id = 1,
        .name = "value",
        .tpe = SFunc.unary(.box, .long),
        .cost_kind = .{ .fixed = 1 },
    };

    pub const PROPOSITION_BYTES = SMethod{
        .method_id = 2,
        .name = "propositionBytes",
        .tpe = SFunc.unary(.box, .{ .coll = &SType.byte }),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const BYTES = SMethod{
        .method_id = 3,
        .name = "bytes",
        .tpe = SFunc.unary(.box, .{ .coll = &SType.byte }),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const ID = SMethod{
        .method_id = 5,
        .name = "id",
        .tpe = SFunc.unary(.box, .{ .coll = &SType.byte }),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const CREATION_INFO = SMethod{
        .method_id = 6,
        .name = "creationInfo",
        .tpe = SFunc.unary(.box, .{ .tuple = &[_]SType{ .int, .{ .coll = &SType.byte } } }),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const TOKENS = SMethod{
        .method_id = 8,
        .name = "tokens",
        .tpe = SFunc.unary(.box, .{
            .coll = &.{ .tuple = &[_]SType{ .{ .coll = &SType.byte }, .long } },
        }),
        .cost_kind = .{ .fixed = 15 },
    };

    // Register access: R0-R9
    pub const R0 = SMethod{ .method_id = 10, .name = "R0", ... };
    pub const R1 = SMethod{ .method_id = 11, .name = "R1", ... };
    pub const R2 = SMethod{ .method_id = 12, .name = "R2", ... };
    pub const R3 = SMethod{ .method_id = 13, .name = "R3", ... };
    pub const R4 = SMethod{ .method_id = 14, .name = "R4", ... };
    pub const R5 = SMethod{ .method_id = 15, .name = "R5", ... };
    pub const R6 = SMethod{ .method_id = 16, .name = "R6", ... };
    pub const R7 = SMethod{ .method_id = 17, .name = "R7", ... };
    pub const R8 = SMethod{ .method_id = 18, .name = "R8", ... };
    pub const R9 = SMethod{ .method_id = 19, .name = "R9", ... };
};
```

## Context Methods

Access transaction context[^9][^10]:

```zig
const SContextMethods = struct {
    pub const TYPE_CODE = 101;

    pub const DATA_INPUTS = SMethod{
        .method_id = 1,
        .name = "dataInputs",
        .tpe = SFunc.unary(.context, .{ .coll = &SType.box }),
        .cost_kind = .{ .fixed = 15 },
    };

    pub const HEADERS = SMethod{
        .method_id = 2,
        .name = "headers",
        .tpe = SFunc.unary(.context, .{ .coll = &SType.header }),
        .cost_kind = .{ .fixed = 15 },
    };

    pub const PRE_HEADER = SMethod{
        .method_id = 3,
        .name = "preHeader",
        .tpe = SFunc.unary(.context, .pre_header),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const INPUTS = SMethod{
        .method_id = 4,
        .name = "INPUTS",
        .tpe = SFunc.unary(.context, .{ .coll = &SType.box }),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const OUTPUTS = SMethod{
        .method_id = 5,
        .name = "OUTPUTS",
        .tpe = SFunc.unary(.context, .{ .coll = &SType.box }),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const HEIGHT = SMethod{
        .method_id = 6,
        .name = "HEIGHT",
        .tpe = SFunc.unary(.context, .int),
        .cost_kind = .{ .fixed = 26 },
    };

    pub const SELF = SMethod{
        .method_id = 7,
        .name = "SELF",
        .tpe = SFunc.unary(.context, .box),
        .cost_kind = .{ .fixed = 10 },
    };

    pub const GET_VAR = SMethod{
        .method_id = 8,
        .name = "getVar",
        .tpe = SFunc.new(&[_]SType{ .context, .byte }, .{ .option = .{ .type_var = "T" } }),
        .cost_kind = .dynamic,
    };
};
```

## GroupElement Methods

Elliptic curve operations[^11][^12]:

```zig
const SGroupElementMethods = struct {
    pub const TYPE_CODE = 7;

    pub const GET_ENCODED = SMethod{
        .method_id = 2,
        .name = "getEncoded",
        .tpe = SFunc.unary(.group_element, .{ .coll = &SType.byte }),
        .cost_kind = .{ .fixed = 250 },
    };

    pub const EXP = SMethod{
        .method_id = 3,
        .name = "exp",
        .tpe = SFunc.binary(.group_element, .big_int, .group_element),
        .cost_kind = .{ .fixed = 900 },
    };

    pub const MULTIPLY = SMethod{
        .method_id = 4,
        .name = "multiply",
        .tpe = SFunc.binary(.group_element, .group_element, .group_element),
        .cost_kind = .{ .fixed = 40 },
    };

    pub const NEGATE = SMethod{
        .method_id = 5,
        .name = "negate",
        .tpe = SFunc.unary(.group_element, .group_element),
        .cost_kind = .{ .fixed = 45 },
    };
};
```

## SigmaProp Methods

Cryptographic proposition operations[^13]:

```zig
const SSigmaPropMethods = struct {
    pub const TYPE_CODE = 8;

    pub const PROP_BYTES = SMethod{
        .method_id = 1,
        .name = "propBytes",
        .tpe = SFunc.unary(.sigma_prop, .{ .coll = &SType.byte }),
        .cost_kind = .{ .fixed = 35 },
    };

    pub const IS_PROVEN = SMethod{
        .method_id = 2,
        .name = "isProven",
        .tpe = SFunc.unary(.sigma_prop, .boolean),
        .cost_kind = .{ .fixed = 10 },
        .deprecated = true, // Use in scripts only
    };
};
```

## Method Resolution

Method lookup by type code and method ID:

```zig
pub fn resolveMethod(type_code: u8, method_id: u8) ?*const SMethod {
    const container = switch (type_code) {
        2, 3, 4, 5 => &SNumericMethods,  // Byte, Short, Int, Long
        6 => &SBigIntMethods,
        7 => &SGroupElementMethods,
        8 => &SSigmaPropMethods,
        9 => &SUnsignedBigIntMethods,     // v6+
        12...23 => &SCollectionMethods,   // Coll[T]
        36...47 => &SOptionMethods,       // Option[T]
        99 => &SBoxMethods,
        100 => &SAvlTreeMethods,
        101 => &SContextMethods,
        104 => &SHeaderMethods,
        105 => &SPreHeaderMethods,
        106 => &SGlobalMethods,
        else => return null,
    };
    return container.getMethodById(method_id);
}
```

## Method Call Evaluation

```zig
const MethodCall = struct {
    receiver_type: SType,
    method: *const SMethod,
    receiver: *const Value,
    args: []const *Value,

    pub fn eval(self: *const MethodCall, env: *const DataEnv, E: *Evaluator) !Any {
        // Evaluate receiver
        const recv = try self.receiver.eval(env, E);

        // Evaluate arguments
        var arg_values = try E.allocator.alloc(Any, self.args.len);
        for (self.args, 0..) |arg, i| {
            arg_values[i] = try arg.eval(env, E);
        }

        // Add cost
        E.addCost(self.method.cost_kind, self.method.op_code);

        // Dispatch to method implementation
        return try self.method.eval(recv, arg_values, E);
    }
};
```

## Summary

This chapter covered the method system that extends ErgoTree types with rich APIs:

- **`MethodsContainer`** organizes methods per type, with each method having a unique ID (1-255) within its container
- **Method resolution** uses the receiver's type code and the method ID to locate the implementation, avoiding opcode space consumption
- **Numeric methods** provide type conversions (`toByte`, `toInt`, `toLong`, `toBigInt`) shared across all numeric types, with v6 adding bitwise operations and byte representation
- **Collection methods** form the richest API with transformation (`map`, `filter`, `fold`), predicates (`exists`, `forall`), and combination operations (`append`, `slice`, `zip`)
- **Box methods** access UTXO properties: `value` (nanoERGs), `tokens`, `propositionBytes`, and registers R0-R9
- **Context methods** provide access to transaction data: `INPUTS`, `OUTPUTS`, `HEIGHT`, `SELF`, `dataInputs`, `headers`, and context variables via `getVar`
- **Cryptographic methods** on `GroupElement` support elliptic curve operations (`exp`, `multiply`, `negate`) and `SigmaProp` provides `propBytes` for serialization

---

*Next: [Chapter 7: Serialization Framework](../part3/ch07-serialization-framework.md)*

[^1]: Scala: [`methods.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala)

[^2]: Rust: [`smethod.rs:36-99`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/smethod.rs#L36-L99) (SMethod, SMethodDesc)

[^3]: Scala: [`methods.scala:232-500`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala#L232-L500)

[^4]: Rust: [`snumeric.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/snumeric.rs)

[^5]: Scala: [`methods.scala:805-1260`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala#L805-L1260)

[^6]: Rust: [`scoll.rs:22-266`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/scoll.rs#L22-L266) (METHOD_DESC, method IDs)

[^7]: Scala: [`methods.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala) (SBoxMethods)

[^8]: Rust: [`sbox.rs:29-92`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/sbox.rs#L29-L92) (VALUE_METHOD, GET_REG_METHOD, TOKENS_METHOD)

[^9]: Scala: [`methods.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala) (SContextMethods)

[^10]: Rust: [`scontext.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/scontext.rs)

[^11]: Scala: [`methods.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala) (SGroupElementMethods)

[^12]: Rust: [`sgroup_elem.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/sgroup_elem.rs)

[^13]: Scala: [`methods.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala) (SSigmaPropMethods)
