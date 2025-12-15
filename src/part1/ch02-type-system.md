# Chapter 2: Type System

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Basic type system concepts (generic types, type hierarchies)
- Prior chapters: [Chapter 1](./ch01-introduction.md)

## Learning Objectives

- List and describe all ErgoTree primitive types
- Explain type code encoding for serialization
- Distinguish embeddable from non-embeddable types
- Work with collection, option, and tuple types

## Type System Overview

ErgoTree uses a **static type system** with[^1][^2]:

- **Structural typing**: Types defined by structure, not names
- **Type inference**: Many types inferred automatically
- **Generic types**: Parameterized collections and options
- **Type codes**: Unique byte codes for serialization

```zig
/// Base type descriptor
const SType = union(enum) {
    // Primitives (embeddable, codes 1-9)
    boolean,
    byte,
    short,
    int,
    long,
    big_int,
    group_element,
    sigma_prop,
    unsigned_big_int, // v6+

    // Compound types
    coll: *const SType,
    option: *const SType,
    tuple: []const SType,
    func: SFunc,

    // Object types (codes 99-106)
    box,
    avl_tree,
    context,
    header,
    pre_header,
    global,

    // Special
    unit,
    any,
    type_var: []const u8,

    pub fn typeCode(self: SType) u8 {
        return switch (self) {
            .boolean => 1,
            .byte => 2,
            .short => 3,
            .int => 4,
            .long => 5,
            .big_int => 6,
            .group_element => 7,
            .sigma_prop => 8,
            .unsigned_big_int => 9,
            .coll => 12,
            .option => 36,
            .tuple => 96,
            .box => 99,
            .avl_tree => 100,
            .context => 101,
            .header => 104,
            .pre_header => 105,
            .global => 106,
            else => 0,
        };
    }

    pub fn isEmbeddable(self: SType) bool {
        return self.typeCode() >= 1 and self.typeCode() <= 9;
    }

    pub fn isNumeric(self: SType) bool {
        return switch (self) {
            .byte, .short, .int, .long, .big_int, .unsigned_big_int => true,
            else => false,
        };
    }
};
```

## Type Hierarchy

```
                              SType
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
    SEmbeddable            SCollection            SOption
         │                 (elemType)            (elemType)
    ┌────┴────┬─────────────────┐
    │         │                 │
SNumericType SBoolean    SGroupElement
    │                     SSigmaProp
    │
    ├── SByte (code 2)
    ├── SShort (code 3)
    ├── SInt (code 4)
    ├── SLong (code 5)
    ├── SBigInt (code 6)
    └── SUnsignedBigInt (code 9, v6+)

Object Types (non-embeddable):
  SBox(99), SAvlTree(100), SContext(101),
  SHeader(104), SPreHeader(105), SGlobal(106)
```

## Primitive Types

### Numeric Types

All numeric types support conversion via `upcast` (widening) and `downcast` (narrowing, throws on overflow)[^3][^4]:

| Type | Code | Size | Range |
|------|------|------|-------|
| `SByte` | 2 | 8-bit | -128 to 127 |
| `SShort` | 3 | 16-bit | -32,768 to 32,767 |
| `SInt` | 4 | 32-bit | ±2.1 billion |
| `SLong` | 5 | 64-bit | ±9.2 quintillion |
| `SBigInt` | 6 | 256-bit | Signed arbitrary |
| `SUnsignedBigInt` | 9 | 256-bit | Unsigned (v6+) |

```zig
const SNumericType = struct {
    type_code: u8,
    numeric_index: u8, // 0=Byte, 1=Short, 2=Int, 3=Long, 4=BigInt, 5=UBigInt

    /// Ordering: Byte < Short < Int < Long < BigInt < UnsignedBigInt
    pub fn canUpcastTo(self: SNumericType, target: SNumericType) bool {
        return self.numeric_index <= target.numeric_index;
    }

    /// Downcast with overflow check
    pub fn downcast(comptime T: type, value: anytype) !T {
        const min = std.math.minInt(T);
        const max = std.math.maxInt(T);
        if (value < min or value > max) {
            return error.ArithmeticOverflow;
        }
        return @intCast(value);
    }
};

// Type instances
const SByte = SNumericType{ .type_code = 2, .numeric_index = 0 };
const SShort = SNumericType{ .type_code = 3, .numeric_index = 1 };
const SInt = SNumericType{ .type_code = 4, .numeric_index = 2 };
const SLong = SNumericType{ .type_code = 5, .numeric_index = 3 };
const SBigInt = SNumericType{ .type_code = 6, .numeric_index = 4 };
const SUnsignedBigInt = SNumericType{ .type_code = 9, .numeric_index = 5 };
```

### Boolean Type

```zig
const SBoolean = struct {
    pub const type_code: u8 = 1;
    pub const is_embeddable = true;
};
```

### Cryptographic Types

**GroupElement** — Point on secp256k1 curve (33 bytes compressed)[^5]:

```zig
const SGroupElement = struct {
    pub const type_code: u8 = 7;

    /// 33 bytes: 1-byte prefix (0x02/0x03) + 32-byte X coordinate
    pub const SERIALIZED_SIZE = 33;
};
```

**SigmaProp** — Cryptographic proposition (required return type)[^6]:

```zig
const SSigmaProp = struct {
    pub const type_code: u8 = 8;

    /// Maximum serialized size
    pub const MAX_SIZE_BYTES: usize = 1024;
};
```

## Type Codes

Type code space partitioning[^7]:

| Range | Description |
|-------|-------------|
| 1-9 | Primitive embeddable types |
| 10-11 | Reserved |
| 12-23 | `Coll[T]` (T primitive) |
| 24-35 | `Coll[Coll[T]]` |
| 36-47 | `Option[T]` |
| 48-59 | `Option[Coll[T]]` |
| 60+ | Other types |

```zig
const TypeCodes = struct {
    // Primitives
    pub const BOOLEAN: u8 = 1;
    pub const BYTE: u8 = 2;
    pub const SHORT: u8 = 3;
    pub const INT: u8 = 4;
    pub const LONG: u8 = 5;
    pub const BIGINT: u8 = 6;
    pub const GROUP_ELEMENT: u8 = 7;
    pub const SIGMA_PROP: u8 = 8;
    pub const UNSIGNED_BIGINT: u8 = 9;

    // Type constructor bases
    pub const PRIM_RANGE: u8 = 12; // MaxPrimTypeCode + 1
    pub const COLL_BASE: u8 = 12;
    pub const NESTED_COLL_BASE: u8 = 24;
    pub const OPTION_BASE: u8 = 36;
    pub const OPTION_COLL_BASE: u8 = 48;

    // Object types
    pub const TUPLE: u8 = 96;
    pub const ANY: u8 = 97;
    pub const UNIT: u8 = 98;
    pub const BOX: u8 = 99;
    pub const AVL_TREE: u8 = 100;
    pub const CONTEXT: u8 = 101;
    pub const HEADER: u8 = 104;
    pub const PREHEADER: u8 = 105;
    pub const GLOBAL: u8 = 106;
};
```

## Embeddable Types

Embeddable types combine with type constructors for single-byte encoding[^8]:

```zig
/// Embed primitive type code into constructor
pub fn embedType(type_constr_base: u8, prim_type_code: u8) u8 {
    return type_constr_base + prim_type_code;
}

// Examples:
// Coll[Byte]  = 12 + 2 = 14
// Coll[Int]   = 12 + 4 = 16
// Option[Long] = 36 + 5 = 41
// Option[Coll[Byte]] = 48 + 2 = 50
```

| Type | Code | Coll[T] | Option[T] |
|------|------|---------|-----------|
| Boolean | 1 | 13 | 37 |
| Byte | 2 | 14 | 38 |
| Short | 3 | 15 | 39 |
| Int | 4 | 16 | 40 |
| Long | 5 | 17 | 41 |
| BigInt | 6 | 18 | 42 |
| GroupElement | 7 | 19 | 43 |
| SigmaProp | 8 | 20 | 44 |
| UnsignedBigInt | 9 | 21 | 45 |

## Collection Types

Collections are homogeneous sequences[^9][^10]:

```zig
const SCollection = struct {
    elem_type: *const SType,

    pub fn typeCode(self: SCollection) u8 {
        if (self.elem_type.isEmbeddable()) {
            return TypeCodes.COLL_BASE + self.elem_type.typeCode();
        }
        return TypeCodes.COLL_BASE; // Followed by element type
    }
};

// Pre-defined collection types (avoid allocation)
const SByteArray = SCollection{ .elem_type = &SType.byte };
const SIntArray = SCollection{ .elem_type = &SType.int };
const SBooleanArray = SCollection{ .elem_type = &SType.boolean };
const SBoxArray = SCollection{ .elem_type = &SType.box };
```

## Option Types

Optional values[^11]:

```zig
const SOption = struct {
    elem_type: *const SType,

    pub fn typeCode(self: SOption) u8 {
        if (self.elem_type.isEmbeddable()) {
            return TypeCodes.OPTION_BASE + self.elem_type.typeCode();
        }
        return TypeCodes.OPTION_BASE;
    }
};

// Pre-defined option types
const SByteOption = SOption{ .elem_type = &SType.byte };
const SIntOption = SOption{ .elem_type = &SType.int };
const SLongOption = SOption{ .elem_type = &SType.long };
const SBoxOption = SOption{ .elem_type = &SType.box };
```

## Tuple Types

Heterogeneous fixed-size sequences:

```zig
const STuple = struct {
    items: []const SType,

    pub const type_code: u8 = 96;

    pub fn pair(left: SType, right: SType) STuple {
        return STuple{ .items = &[_]SType{ left, right } };
    }
};
```

## Function Types

Function signatures for lambdas and methods:

```zig
const SFunc = struct {
    t_dom: []const SType,     // Domain (argument types)
    t_range: *const SType,    // Range (return type)
    tpe_params: []const STypeVar, // Generic type parameters

    pub const type_code: u8 = 246;
};

// Example: (Int) => Boolean
const intToBool = SFunc{
    .t_dom = &[_]SType{SType.int},
    .t_range = &SType.boolean,
    .tpe_params = &[_]STypeVar{},
};
```

## Object Types

| Type | Code | Description |
|------|------|-------------|
| `SBox` | 99 | UTXO with value, script, tokens, registers |
| `SAvlTree` | 100 | Authenticated dictionary (Merkle proofs) |
| `SContext` | 101 | Transaction context |
| `SHeader` | 104 | Block header |
| `SPreHeader` | 105 | Pre-solved block header |
| `SGlobal` | 106 | Global operations |

## Type Variables

Used internally by compiler for generic methods (never serialized)[^12]:

```zig
const STypeVar = struct {
    name: []const u8,

    // Standard type variables
    pub const T = STypeVar{ .name = "T" };
    pub const R = STypeVar{ .name = "R" };
    pub const K = STypeVar{ .name = "K" };
    pub const V = STypeVar{ .name = "V" };
    pub const IV = STypeVar{ .name = "IV" }; // Input Value
    pub const OV = STypeVar{ .name = "OV" }; // Output Value
};
```

## Version Differences

v6 additions[^13]:

- `SUnsignedBigInt` (type code 9)
- Bitwise operations on numeric types
- Additional numeric methods (`toBytes`, `toBits`, shifts)

```zig
pub fn allPredefTypes(version: ErgoTreeVersion) []const SType {
    const v5_types = &[_]SType{
        .boolean, .byte, .short, .int, .long, .big_int,
        .context, .global, .header, .pre_header, .avl_tree,
        .group_element, .sigma_prop, .box, .unit, .any,
    };

    if (version.value >= 3) { // v6+
        return v5_types ++ &[_]SType{.unsigned_big_int};
    }
    return v5_types;
}
```

## Complete Type Code Reference

| Type | Code | Embeddable |
|------|------|------------|
| Boolean | 1 | Yes |
| Byte | 2 | Yes |
| Short | 3 | Yes |
| Int | 4 | Yes |
| Long | 5 | Yes |
| BigInt | 6 | Yes |
| GroupElement | 7 | Yes |
| SigmaProp | 8 | Yes |
| UnsignedBigInt | 9 | Yes |
| Coll[T] | 12 | Constructor |
| Option[T] | 36 | Constructor |
| Tuple | 96 | No |
| Any | 97 | No |
| Unit | 98 | No |
| Box | 99 | No |
| AvlTree | 100 | No |
| Context | 101 | No |
| Header | 104 | No |
| PreHeader | 105 | No |
| Global | 106 | No |

## Summary

- Every type has a unique **type code** for serialization
- **Embeddable types** (codes 1-9) combine with constructors for single-byte encoding
- Numeric types form an ordered hierarchy with exact conversions
- **SigmaProp** is the required return type for all scripts
- v6 adds `SUnsignedBigInt` and enhanced numeric operations

---

*Next: [Chapter 3: ErgoTree Structure](./ch03-ergotree-structure.md)*

[^1]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:17-61`

[^2]: Rust: `ergotree-ir/src/types/stype.rs:27-76`

[^3]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:395-575` (numeric type definitions)

[^4]: Rust: `ergotree-ir/src/types/snumeric.rs:12-37` (method IDs)

[^5]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala` (SGroupElement definition)

[^6]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala` (SSigmaProp definition)

[^7]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:320-332` (type code ranges)

[^8]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:305-313` (SEmbeddable trait)

[^9]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:743-799` (SCollection)

[^10]: Rust: `ergotree-ir/src/types/scoll.rs`

[^11]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:691-741` (SOption)

[^12]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:67-95` (type variables)

[^13]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala:105-128` (version differences)