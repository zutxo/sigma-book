# Appendix A: Complete Type Code Table

Complete reference for all type codes used in ErgoTree serialization[^1][^2].

## Type Code Ranges

```
Type Code Organization
══════════════════════════════════════════════════════════════════

Range          Usage
─────────────────────────────────────────────────────────────────
0x00           Reserved (invalid)
0x01-0x09      Primitive types (embeddable)
0x0A-0x0B      Reserved
0x0C           Collection type constructor
0x0D-0x17      Reserved
0x18           Nested collection (Coll[Coll[T]])
0x19-0x23      Reserved
0x24           Option type constructor
0x25-0x3B      Reserved
0x3C-0x5F      Pair type constructors
0x60           Tuple type constructor
0x61-0x6A      Object types (non-embeddable)
0x6B-0x6F      Reserved for future object types
```

## Primitive Types (Embeddable)

| Dec | Hex  | Type            | Size     | Zig Type             |
|-----|------|-----------------|----------|----------------------|
| 1   | 0x01 | SBoolean        | 1 bit    | `bool`               |
| 2   | 0x02 | SByte           | 8 bits   | `i8`                 |
| 3   | 0x03 | SShort          | 16 bits  | `i16`                |
| 4   | 0x04 | SInt            | 32 bits  | `i32`                |
| 5   | 0x05 | SLong           | 64 bits  | `i64`                |
| 6   | 0x06 | SBigInt         | 256 bits | `BigInt256`          |
| 7   | 0x07 | SGroupElement   | 33 bytes | `Ecp` (compressed)   |
| 8   | 0x08 | SSigmaProp      | variable | `SigmaBoolean`       |
| 9   | 0x09 | SUnsignedBigInt | 256 bits | `UnsignedBigInt256`  |

## Object Types

| Dec | Hex  | Type       | Description                    |
|-----|------|------------|--------------------------------|
| 97  | 0x61 | SAny       | Supertype of all types         |
| 98  | 0x62 | SUnit      | Unit type (singleton)          |
| 99  | 0x63 | SBox       | Transaction box                |
| 100 | 0x64 | SAvlTree   | Authenticated dictionary       |
| 101 | 0x65 | SContext   | Execution context              |
| 102 | 0x66 | SString    | String (ErgoScript only)       |
| 103 | 0x67 | STypeVar   | Type variable (internal)       |
| 104 | 0x68 | SHeader    | Block header                   |
| 105 | 0x69 | SPreHeader | Block pre-header               |
| 106 | 0x6A | SGlobal    | Global object (SigmaDslBuilder)|

## Type Constructors

| Dec | Hex  | Constructor          | Example              | Serialized As      |
|-----|------|----------------------|----------------------|--------------------|
| 12  | 0x0C | SColl                | `Coll[Byte]`         | `0x0C 0x02`        |
| 24  | 0x18 | Nested SColl         | `Coll[Coll[Int]]`    | `0x18 0x04`        |
| 36  | 0x24 | SOption              | `Option[Long]`       | `0x24 0x05`        |
| 60  | 0x3C | Pair (first generic) | `(_, Byte)`          | `0x3C 0x02`        |
| 72  | 0x48 | Pair (second generic)| `(Int, _)`           | `0x48 0x04`        |
| 84  | 0x54 | Pair (symmetric)     | `(Long, Long)`       | `0x54 0x05`        |
| 96  | 0x60 | STuple               | `(Int, Boolean, ...)` | `0x60 len types...`|

## Zig Type Definition

```zig
const TypeCode = enum(u8) {
    // Primitive types
    boolean = 0x01,
    byte = 0x02,
    short = 0x03,
    int = 0x04,
    long = 0x05,
    big_int = 0x06,
    group_element = 0x07,
    sigma_prop = 0x08,
    unsigned_big_int = 0x09,

    // Type constructors
    coll = 0x0C,
    nested_coll = 0x18,
    option = 0x24,
    pair_first = 0x3C,
    pair_second = 0x48,
    pair_symmetric = 0x54,
    tuple = 0x60,

    // Object types
    any = 0x61,
    unit = 0x62,
    box = 0x63,
    avl_tree = 0x64,
    context = 0x65,
    string = 0x66,
    type_var = 0x67,
    header = 0x68,
    pre_header = 0x69,
    global = 0x6A,

    pub fn isPrimitive(self: TypeCode) bool {
        return @intFromEnum(self) >= 0x01 and @intFromEnum(self) <= 0x09;
    }

    pub fn isEmbeddable(self: TypeCode) bool {
        return self.isPrimitive();
    }

    pub fn isNumeric(self: TypeCode) bool {
        return switch (self) {
            .byte, .short, .int, .long, .big_int, .unsigned_big_int => true,
            else => false,
        };
    }
};
```

## Type Serialization

```zig
const SType = union(enum) {
    boolean,
    byte,
    short,
    int,
    long,
    big_int,
    group_element,
    sigma_prop,
    unsigned_big_int,
    coll: *const SType,
    option: *const SType,
    tuple: []const SType,
    box,
    avl_tree,
    context,
    header,
    pre_header,
    global,
    unit,
    any,

    pub fn typeCode(self: *const SType) u8 {
        return switch (self.*) {
            .boolean => 0x01,
            .byte => 0x02,
            .short => 0x03,
            .int => 0x04,
            .long => 0x05,
            .big_int => 0x06,
            .group_element => 0x07,
            .sigma_prop => 0x08,
            .unsigned_big_int => 0x09,
            .coll => |elem| blk: {
                if (elem.* == .coll) break :blk 0x18;
                break :blk 0x0C;
            },
            .option => 0x24,
            .tuple => 0x60,
            .box => 0x63,
            .avl_tree => 0x64,
            .context => 0x65,
            .header => 0x68,
            .pre_header => 0x69,
            .global => 0x6A,
            .unit => 0x62,
            .any => 0x61,
        };
    }
};
```

## Encoding Rules

```
Type Encoding Examples
══════════════════════════════════════════════════════════════════

Simple Types:
  SInt           → [0x04]
  SBoolean       → [0x01]
  SGroupElement  → [0x07]

Collections:
  Coll[Byte]         → [0x0C, 0x02]      (coll + byte)
  Coll[Int]          → [0x0C, 0x04]      (coll + int)
  Coll[Coll[Byte]]   → [0x18, 0x02]      (nested_coll + byte)

Options:
  Option[Int]        → [0x24, 0x04]      (option + int)
  Option[Box]        → [0x24, 0x63]      (option + box)

Tuples (2 elements):
  (Int, Int)         → [0x54, 0x04]      (symmetric + int)
  (Int, Long)        → [0x48, 0x04, 0x05] (pair2 + int + long)
  (Long, Int)        → [0x3C, 0x05, 0x04] (pair1 + long + int)

Tuples (3+ elements):
  (Int, Long, Byte)  → [0x60, 0x03, 0x04, 0x05, 0x02]
                       (tuple + len + int + long + byte)
```

## Constants

```zig
const TypeConstants = struct {
    /// First type code for primitive types
    pub const FIRST_PRIMITIVE_TYPE: u8 = 0x01;
    /// Last type code for primitive types
    pub const LAST_PRIMITIVE_TYPE: u8 = 0x09;
    /// Maximum supported type code
    pub const MAX_TYPE_CODE: u8 = 0x6A;
    /// Last data type (can be serialized as data)
    pub const LAST_DATA_TYPE: u8 = 111;
};
```

---

*[Previous: Chapter 31](../part10/ch31-performance-engineering.md) | [Next: Appendix B](./appendix-b-opcodes.md)*

[^1]: Scala: `core/shared/src/main/scala/sigma/ast/SType.scala`

[^2]: Rust: `ergotree-ir/src/types/stype.rs:28-76`
