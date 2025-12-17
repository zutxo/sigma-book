# Chapter 7: Serialization Framework

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Binary encoding concepts (bits, bytes, big-endian vs little-endian)
- Familiarity with variable-length encoding techniques and their space-efficiency trade-offs
- Prior chapters: [Chapter 2](../part1/ch02-type-system.md) for type codes, [Chapter 5](../part2/ch05-operations-opcodes.md) for opcodes

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain VLQ (Variable-Length Quantity) encoding and how it achieves compact integer representation
- Describe ZigZag encoding and why it improves VLQ efficiency for signed integers
- Implement type serialization using the type code embedding scheme
- Use `SigmaByteReader` and `SigmaByteWriter` for type-aware serialization

## Serialization Architecture

Blockchain storage is expensive—every byte of an ErgoTree increases transaction fees and network bandwidth. The serialization framework therefore prioritizes compactness while maintaining determinism (identical inputs must produce identical outputs across all implementations). The system uses a layered design where each layer handles a specific concern[^1][^2]:

```
┌─────────────────────────────────────────────────┐
│              Application Layer                  │
│      (ErgoTree, Box, Transaction)               │
├─────────────────────────────────────────────────┤
│            Value Serializers                    │
│      (ConstantSerializer, MethodCall)           │
├─────────────────────────────────────────────────┤
│         SigmaByteReader/Writer                  │
│      (Type-aware, constant store)               │
├─────────────────────────────────────────────────┤
│           VLQ Encoding Layer                    │
│      (Variable-length integers)                 │
├─────────────────────────────────────────────────┤
│            Byte Buffer I/O                      │
│      (Raw read/write operations)                │
└─────────────────────────────────────────────────┘
```

### Base Serializer Interface

```zig
const SigmaSerializer = struct {
    pub const MAX_PROPOSITION_SIZE: usize = 4096;
    pub const MAX_TREE_DEPTH: u32 = 110;

    pub fn toBytes(comptime T: type, obj: T, allocator: Allocator) ![]u8 {
        var list = std.ArrayList(u8).init(allocator);
        var writer = SigmaByteWriter.init(&list);
        try T.serialize(obj, &writer);
        return list.toOwnedSlice();
    }

    pub fn fromBytes(comptime T: type, bytes: []const u8) !T {
        var reader = SigmaByteReader.init(bytes);
        return try T.deserialize(&reader);
    }
};
```

## VLQ Encoding

Variable-Length Quantity (VLQ) represents integers compactly[^3][^4]:

```
Value Range              Bytes   Format
─────────────────────────────────────────────────
0 - 127                  1       0xxxxxxx
128 - 16,383             2       1xxxxxxx 0xxxxxxx
16,384 - 2,097,151       3       1xxxxxxx 1xxxxxxx 0xxxxxxx
2,097,152 - 268,435,455  4       1xxxxxxx 1xxxxxxx 1xxxxxxx 0xxxxxxx
...                      ...     ...
```

Each byte uses 7 bits for data; MSB is continuation flag:
- `0` = final byte
- `1` = more bytes follow

### VLQ Implementation

```zig
const VlqEncoder = struct {
    /// Write unsigned integer using VLQ encoding
    pub fn putUInt(writer: anytype, value: u64) !void {
        var v = value;
        while ((v & 0xFFFFFF80) != 0) {
            try writer.writeByte(@intCast((v & 0x7F) | 0x80));
            v >>= 7;
        }
        try writer.writeByte(@intCast(v & 0x7F));
    }

    /// Read unsigned integer using VLQ decoding
    /// Maximum 10 bytes for u64 (ceil(64/7) = 10)
    pub fn getUInt(reader: anytype) !u64 {
        const MAX_VLQ_BYTES: u6 = 10;  // ceil(64/7) = 10 bytes max
        var result: u64 = 0;
        var shift: u6 = 0;
        var byte_count: u6 = 0;
        while (shift < 64) {
            const b = try reader.readByte();
            byte_count += 1;
            if (byte_count > MAX_VLQ_BYTES) return error.VlqTooLong;
            result |= @as(u64, b & 0x7F) << shift;
            if ((b & 0x80) == 0) return result;
            shift += 7;
        }
        return error.VlqDecodingFailed;
    }
};
// NOTE: In production, VLQ decoding should use compile-time assertions to
// verify max byte counts. See ZIGMA_STYLE.md for bounded iteration patterns.
```

### VLQ Size by Value Range

```
Unsigned Value           Bytes
─────────────────────────────────
0 - 127                  1
128 - 16,383             2
16,384 - 2,097,151       3
2,097,152 - 268M         4
268M - 34B               5
34B - 4T                 6
4T - 562T                7
562T - 72P               8
72P - 9E                 9
> 9E                     10
```

## ZigZag Encoding

VLQ assumes non-negative values—it encodes the magnitude directly. For signed integers like `-1`, the two's complement representation has all high bits set, resulting in maximum VLQ length. ZigZag encoding solves this by mapping signed values to unsigned in a way that preserves magnitude: small positive and negative numbers both produce small unsigned values[^5][^6]:

```
Signed    ZigZag Encoded
────────────────────────
 0        0
-1        1
 1        2
-2        3
 2        4
-3        5
 n        2n     (n >= 0)
-n        2n-1   (n < 0)
```

### ZigZag Implementation

```zig
const ZigZag = struct {
    /// Encode signed 32-bit to unsigned
    pub fn encode32(n: i32) u64 {
        // Arithmetic right shift replicates sign bit
        return @bitCast(@as(i64, (n << 1) ^ (n >> 31)));
    }

    /// Decode unsigned back to signed 32-bit
    pub fn decode32(n: u64) i32 {
        const v: u32 = @intCast(n);
        return @as(i32, @intCast(v >> 1)) ^ -@as(i32, @intCast(v & 1));
    }

    /// Encode signed 64-bit to unsigned
    pub fn encode64(n: i64) u64 {
        return @bitCast((n << 1) ^ (n >> 63));
    }

    /// Decode unsigned back to signed 64-bit
    pub fn decode64(n: u64) i64 {
        return @as(i64, @intCast(n >> 1)) ^ -@as(i64, @intCast(n & 1));
    }
};
```

ZigZag ensures small-magnitude signed values use few bytes:

```
Value     ZigZag    VLQ Bytes
─────────────────────────────
 0        0         1
-1        1         1
 1        2         1
-64       127       1
 64       128       2
-65       129       2
```

## SigmaByteWriter

The writer handles type-aware serialization with cost tracking[^7][^8]:

```zig
const SigmaByteWriter = struct {
    buffer: *std.ArrayList(u8),
    constant_store: ?*ConstantStore,
    tree_version: ErgoTreeVersion,

    pub fn init(buffer: *std.ArrayList(u8)) SigmaByteWriter {
        return .{
            .buffer = buffer,
            .constant_store = null,
            .tree_version = .v0,
        };
    }

    /// Write single byte
    pub fn putByte(self: *SigmaByteWriter, b: u8) !void {
        try self.buffer.append(b);
    }

    /// Write byte slice
    pub fn putBytes(self: *SigmaByteWriter, bytes: []const u8) !void {
        try self.buffer.appendSlice(bytes);
    }

    /// Write unsigned integer (VLQ encoded)
    pub fn putUInt(self: *SigmaByteWriter, value: u64) !void {
        try VlqEncoder.putUInt(self.buffer.writer(), value);
    }

    /// Write signed short (ZigZag + VLQ)
    pub fn putShort(self: *SigmaByteWriter, value: i16) !void {
        try self.putUInt(ZigZag.encode32(value));
    }

    /// Write signed int (ZigZag + VLQ)
    pub fn putInt(self: *SigmaByteWriter, value: i32) !void {
        try self.putUInt(ZigZag.encode32(value));
    }

    /// Write signed long (ZigZag + VLQ)
    pub fn putLong(self: *SigmaByteWriter, value: i64) !void {
        try self.putUInt(ZigZag.encode64(value));
    }

    /// Write type descriptor
    pub fn putType(self: *SigmaByteWriter, tpe: SType) !void {
        try TypeSerializer.serialize(tpe, self);
    }

    /// Write value with optional constant extraction
    pub fn putValue(self: *SigmaByteWriter, value: *const Value) !void {
        if (self.constant_store) |store| {
            if (value.isConstant()) {
                const idx = store.put(value.asConstant());
                try self.putByte(OpCode.ConstantPlaceholder.value);
                try self.putUInt(idx);
                return;
            }
        }
        try ValueSerializer.serialize(value, self);
    }
};
```

## SigmaByteReader

The reader provides type-aware deserialization[^9][^10]:

```zig
const SigmaByteReader = struct {
    data: []const u8,
    pos: usize,
    constant_store: ConstantStore,
    substitute_placeholders: bool,
    val_def_type_store: ValDefTypeStore,
    tree_version: ErgoTreeVersion,

    pub fn init(data: []const u8) SigmaByteReader {
        return .{
            .data = data,
            .pos = 0,
            .constant_store = ConstantStore.empty(),
            .substitute_placeholders = false,
            .val_def_type_store = ValDefTypeStore.init(),
            .tree_version = .v0,
        };
    }

    pub fn initWithStore(data: []const u8, store: ConstantStore) SigmaByteReader {
        var reader = init(data);
        reader.constant_store = store;
        reader.substitute_placeholders = true;
        return reader;
    }

    /// Read single byte
    pub fn getByte(self: *SigmaByteReader) !u8 {
        if (self.pos >= self.data.len) return error.EndOfStream;
        const b = self.data[self.pos];
        self.pos += 1;
        return b;
    }

    /// Read byte slice
    pub fn getBytes(self: *SigmaByteReader, n: usize) ![]const u8 {
        if (self.pos + n > self.data.len) return error.EndOfStream;
        const slice = self.data[self.pos..][0..n];
        self.pos += n;
        return slice;
    }

    /// Read unsigned integer (VLQ)
    pub fn getUInt(self: *SigmaByteReader) !u64 {
        return VlqEncoder.getUInt(self);
    }

    /// Read signed short (VLQ + ZigZag)
    pub fn getShort(self: *SigmaByteReader) !i16 {
        const v = try self.getUInt();
        return @intCast(ZigZag.decode32(v));
    }

    /// Read signed int (VLQ + ZigZag)
    pub fn getInt(self: *SigmaByteReader) !i32 {
        return ZigZag.decode32(try self.getUInt());
    }

    /// Read signed long (VLQ + ZigZag)
    pub fn getLong(self: *SigmaByteReader) !i64 {
        return ZigZag.decode64(try self.getUInt());
    }

    /// Read type descriptor
    pub fn getType(self: *SigmaByteReader) !SType {
        return TypeSerializer.deserialize(self);
    }

    /// Read value expression
    pub fn getValue(self: *SigmaByteReader) !*Value {
        return ValueSerializer.deserialize(self);
    }

    /// Remaining bytes available
    pub fn remaining(self: *const SigmaByteReader) usize {
        return self.data.len - self.pos;
    }

    // Reader interface for VLQ
    pub fn readByte(self: *SigmaByteReader) !u8 {
        return self.getByte();
    }
};
```

### Constant Store

Manages constants during ErgoTree serialization[^11]:

```zig
const ConstantStore = struct {
    constants: []const Constant,
    extracted: std.ArrayList(Constant),

    pub fn empty() ConstantStore {
        return .{
            .constants = &.{},
            .extracted = undefined,
        };
    }

    pub fn init(constants: []const Constant, allocator: Allocator) ConstantStore {
        return .{
            .constants = constants,
            .extracted = std.ArrayList(Constant).init(allocator),
        };
    }

    /// Get constant by index
    pub fn get(self: *const ConstantStore, index: usize) !Constant {
        if (index >= self.constants.len) return error.IndexOutOfBounds;
        return self.constants[index];
    }

    /// Store constant during extraction, return index
    pub fn put(self: *ConstantStore, c: Constant) !u32 {
        const idx = self.extracted.items.len;
        try self.extracted.append(c);
        return @intCast(idx);
    }
};
```

## Type Serialization

Types use a compact encoding scheme based on type codes[^12][^13]:

```
Type Code Space
───────────────────────────────────────────────────────────
 1-11    Primitive embeddable types
12-23    Coll[primitive]           (12 + primCode)
24-35    Coll[Coll[primitive]]     (24 + primCode)
36-47    Option[primitive]         (36 + primCode)
48-59    Option[Coll[primitive]]   (48 + primCode)
60-71    (primitive, T2) pairs     (60 + primCode)
72-83    (T1, primitive) pairs     (72 + primCode)
84-95    (primitive, primitive)    (84 + primCode) symmetric
96       Tuple (generic)
97-106   Object types (Any, Unit, Box, ...)
112      SFunc (v6+)
```

### Type Code Constants

```zig
const TypeCode = struct {
    value: u8,

    // Primitive types (embeddable)
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
    pub const MAX_PRIM: u8 = 11;
    pub const PRIM_RANGE: u8 = 12;  // MAX_PRIM + 1
    pub const COLL: u8 = 12;
    pub const NESTED_COLL: u8 = 24;
    pub const OPTION: u8 = 36;
    pub const OPTION_COLL: u8 = 48;
    pub const TUPLE_PAIR1: u8 = 60;
    pub const TUPLE_PAIR2: u8 = 72;
    pub const TUPLE_SYMMETRIC: u8 = 84;
    pub const TUPLE: u8 = 96;

    // Object types
    pub const ANY: u8 = 97;
    pub const UNIT: u8 = 98;
    pub const BOX: u8 = 99;
    pub const AVL_TREE: u8 = 100;
    pub const CONTEXT: u8 = 101;
    pub const STRING: u8 = 102;
    pub const TYPE_VAR: u8 = 103;
    pub const HEADER: u8 = 104;
    pub const PRE_HEADER: u8 = 105;
    pub const GLOBAL: u8 = 106;
    pub const FUNC: u8 = 112;

    /// Embed primitive type into container code
    pub fn embed(container_base: u8, prim_code: u8) u8 {
        return container_base + prim_code;
    }

    /// Extract container and primitive from combined code
    pub fn unpack(code: u8) struct { container: ?u8, primitive: ?u8 } {
        if (code >= TUPLE) return .{ .container = null, .primitive = null };
        const container_id = (code / PRIM_RANGE) * PRIM_RANGE;
        const type_id = code % PRIM_RANGE;
        return .{
            .container = if (container_id == 0) null else container_id,
            .primitive = if (type_id == 0) null else type_id,
        };
    }
};
```

### Type Serializer

```zig
const TypeSerializer = struct {
    pub fn serialize(tpe: SType, w: *SigmaByteWriter) !void {
        switch (tpe) {
            // Primitives - single byte
            .boolean => try w.putByte(TypeCode.BOOLEAN),
            .byte => try w.putByte(TypeCode.BYTE),
            .short => try w.putByte(TypeCode.SHORT),
            .int => try w.putByte(TypeCode.INT),
            .long => try w.putByte(TypeCode.LONG),
            .big_int => try w.putByte(TypeCode.BIGINT),
            .group_element => try w.putByte(TypeCode.GROUP_ELEMENT),
            .sigma_prop => try w.putByte(TypeCode.SIGMA_PROP),
            .unsigned_big_int => try w.putByte(TypeCode.UNSIGNED_BIGINT),

            // Object types
            .box => try w.putByte(TypeCode.BOX),
            .avl_tree => try w.putByte(TypeCode.AVL_TREE),
            .context => try w.putByte(TypeCode.CONTEXT),
            .header => try w.putByte(TypeCode.HEADER),
            .pre_header => try w.putByte(TypeCode.PRE_HEADER),
            .global => try w.putByte(TypeCode.GLOBAL),
            .unit => try w.putByte(TypeCode.UNIT),
            .any => try w.putByte(TypeCode.ANY),

            // Collections
            .coll => |elem| {
                if (elem.isEmbeddable()) {
                    // Single byte: Coll[primitive]
                    try w.putByte(TypeCode.embed(TypeCode.COLL, elem.typeCode()));
                } else if (elem.* == .coll) {
                    const inner = elem.coll;
                    if (inner.isEmbeddable()) {
                        // Single byte: Coll[Coll[primitive]]
                        try w.putByte(TypeCode.embed(TypeCode.NESTED_COLL, inner.typeCode()));
                    } else {
                        try w.putByte(TypeCode.COLL);
                        try serialize(elem.*, w);
                    }
                } else {
                    try w.putByte(TypeCode.COLL);
                    try serialize(elem.*, w);
                }
            },

            // Options
            .option => |elem| {
                if (elem.isEmbeddable()) {
                    try w.putByte(TypeCode.embed(TypeCode.OPTION, elem.typeCode()));
                } else if (elem.* == .coll) {
                    const inner = elem.coll;
                    if (inner.isEmbeddable()) {
                        try w.putByte(TypeCode.embed(TypeCode.OPTION_COLL, inner.typeCode()));
                    } else {
                        try w.putByte(TypeCode.OPTION);
                        try serialize(elem.*, w);
                    }
                } else {
                    try w.putByte(TypeCode.OPTION);
                    try serialize(elem.*, w);
                }
            },

            // Tuples (pairs)
            .tuple => |items| {
                if (items.len == 2) {
                    try serializePair(items[0], items[1], w);
                } else {
                    try w.putByte(TypeCode.TUPLE);
                    try w.putByte(@intCast(items.len));
                    for (items) |item| {
                        try serialize(item, w);
                    }
                }
            },

            // Functions (v6+)
            .func => |f| {
                try w.putByte(TypeCode.FUNC);
                try w.putByte(@intCast(f.t_dom.len));
                for (f.t_dom) |arg| try serialize(arg, w);
                try serialize(f.t_range.*, w);
                try w.putByte(@intCast(f.tpe_params.len));
                for (f.tpe_params) |p| {
                    try w.putByte(TypeCode.TYPE_VAR);
                    try w.putBytes(p.name);
                }
            },

            else => return error.UnsupportedType,
        }
    }

    fn serializePair(t1: SType, t2: SType, w: *SigmaByteWriter) !void {
        const e1 = t1.isEmbeddable();
        const e2 = t2.isEmbeddable();

        if (e1 and e2 and std.meta.eql(t1, t2)) {
            // Symmetric pair: (Int, Int)
            try w.putByte(TypeCode.embed(TypeCode.TUPLE_SYMMETRIC, t1.typeCode()));
        } else if (e1) {
            // First is primitive: (Int, T)
            try w.putByte(TypeCode.embed(TypeCode.TUPLE_PAIR1, t1.typeCode()));
            try serialize(t2, w);
        } else if (e2) {
            // Second is primitive: (T, Int)
            try w.putByte(TypeCode.embed(TypeCode.TUPLE_PAIR2, t2.typeCode()));
            try serialize(t1, w);
        } else {
            // Both non-primitive
            try w.putByte(TypeCode.TUPLE_PAIR1);
            try serialize(t1, w);
            try serialize(t2, w);
        }
    }

    pub fn deserialize(r: *SigmaByteReader) !SType {
        const c = try r.getByte();
        return parseWithTag(r, c);
    }

    fn parseWithTag(r: *SigmaByteReader, c: u8) !SType {
        if (c < TypeCode.TUPLE) {
            const unpacked = TypeCode.unpack(c);
            const elem_type = if (unpacked.primitive) |p|
                try getEmbeddableType(p, r.tree_version)
            else
                try deserialize(r);

            if (unpacked.container) |container| {
                return switch (container) {
                    TypeCode.COLL => .{ .coll = &elem_type },
                    TypeCode.NESTED_COLL => .{ .coll = &SType{ .coll = &elem_type } },
                    TypeCode.OPTION => .{ .option = &elem_type },
                    TypeCode.OPTION_COLL => .{ .option = &SType{ .coll = &elem_type } },
                    TypeCode.TUPLE_PAIR1 => blk: {
                        const t2 = try deserialize(r);
                        break :blk .{ .tuple = &[_]SType{ elem_type, t2 } };
                    },
                    TypeCode.TUPLE_PAIR2 => blk: {
                        const t1 = try deserialize(r);
                        break :blk .{ .tuple = &[_]SType{ t1, elem_type } };
                    },
                    TypeCode.TUPLE_SYMMETRIC => .{ .tuple = &[_]SType{ elem_type, elem_type } },
                    else => return error.InvalidTypeCode,
                };
            }
            return elem_type;
        }

        return switch (c) {
            TypeCode.TUPLE => blk: {
                const len = try r.getByte();
                var items: [8]SType = undefined;
                for (0..len) |i| items[i] = try deserialize(r);
                break :blk .{ .tuple = items[0..len] };
            },
            TypeCode.ANY => .any,
            TypeCode.UNIT => .unit,
            TypeCode.BOX => .box,
            TypeCode.AVL_TREE => .avl_tree,
            TypeCode.CONTEXT => .context,
            TypeCode.HEADER => .header,
            TypeCode.PRE_HEADER => .pre_header,
            TypeCode.GLOBAL => .global,
            TypeCode.FUNC => blk: {
                if (r.tree_version.value < 3) return error.UnsupportedVersion;
                const dom_len = try r.getByte();
                var t_dom: [255]SType = undefined;
                for (0..dom_len) |i| t_dom[i] = try deserialize(r);
                const t_range = try deserialize(r);
                // ... parse tpe_params
                break :blk .{ .func = undefined }; // Simplified
            },
            else => error.InvalidTypeCode,
        };
    }

    fn getEmbeddableType(code: u8, version: ErgoTreeVersion) !SType {
        return switch (code) {
            TypeCode.BOOLEAN => .boolean,
            TypeCode.BYTE => .byte,
            TypeCode.SHORT => .short,
            TypeCode.INT => .int,
            TypeCode.LONG => .long,
            TypeCode.BIGINT => .big_int,
            TypeCode.GROUP_ELEMENT => .group_element,
            TypeCode.SIGMA_PROP => .sigma_prop,
            TypeCode.UNSIGNED_BIGINT => blk: {
                if (version.value < 3) return error.UnsupportedVersion;
                break :blk .unsigned_big_int;
            },
            else => error.InvalidTypeCode,
        };
    }
};
```

## Encoding Examples

### Example: Encode 300 as VLQ

```
300 = 0x12C = 0b100101100

Step 1: Take low 7 bits, set continuation: 0x2C | 0x80 = 0xAC
Step 2: Shift right 7: 300 >> 7 = 2
Step 3: Take low 7 bits, no continuation: 0x02

Result: [0xAC, 0x02]
```

### Example: Encode -5 as ZigZag + VLQ

```
ZigZag(-5) = (-5 << 1) ^ (-5 >> 31)
          = -10 ^ -1
          = 9

VLQ(9) = [0x09]  (fits in 7 bits)
```

### Example: Serialize Coll[Int]

```
Coll[Int] → single byte
         → TypeCode.COLL + TypeCode.INT
         → 12 + 4 = 16 = 0x10
```

### Example: Serialize (Int, Long)

```
(Int, Long) → TUPLE_PAIR1 + INT, then Long
           → 60 + 4 = 64, then 5
           → [0x40, 0x05]
```

## Summary

This chapter covered the serialization framework that enables compact, deterministic encoding of ErgoTree structures:

- **VLQ (Variable-Length Quantity) encoding** represents integers using 7 data bits per byte with a continuation flag, achieving compact representation where small values use fewer bytes
- **ZigZag encoding** transforms signed integers to unsigned before VLQ encoding, ensuring small-magnitude values (positive or negative) remain compact
- **Type code embedding** packs common type patterns (like `Coll[Int]` or `Option[Long]`) into single bytes by combining container and primitive codes
- **`SigmaByteWriter`** provides type-aware serialization with optional constant extraction for segregated constant trees
- **`SigmaByteReader`** manages deserialization state including constant stores for placeholder resolution and version tracking
- The **type code space** (0-112) is partitioned to enable single-byte encoding for primitives, nested collections, options, and pairs

---

*Next: [Chapter 8: Value Serializers](./ch08-value-serializers.md)*

[^1]: Scala: [`SigmaSerializer.scala:24-60`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/SigmaSerializer.scala#L24-L60)

[^2]: Rust: [`serializable.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/serializable.rs)

[^3]: Scala: [`VLQByteBufferWriter.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/scorex/util/serialization/VLQByteBufferWriter.scala)

[^4]: Rust: [`vlq_encode.rs:94-112`](https://github.com/ergoplatform/sigma-rust/blob/develop/sigma-ser/src/vlq_encode.rs#L94-L112)

[^5]: Scala: (via scorex-util ZigZag implementation)

[^6]: Rust: [`zig_zag_encode.rs:12-40`](https://github.com/ergoplatform/sigma-rust/blob/develop/sigma-ser/src/zig_zag_encode.rs#L12-L40)

[^7]: Scala: [`SigmaByteWriter.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/SigmaByteWriter.scala)

[^8]: Rust: [`sigma_byte_writer.rs:9-69`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/sigma_byte_writer.rs#L9-L69)

[^9]: Scala: [`SigmaByteReader.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/SigmaByteReader.scala)

[^10]: Rust: [`sigma_byte_reader.rs:12-161`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/sigma_byte_reader.rs#L12-L161)

[^11]: Rust: [`constant_store.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/constant_store.rs)

[^12]: Scala: [`TypeSerializer.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/serialization/TypeSerializer.scala)

[^13]: Rust: [`types.rs:18-160`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/types.rs#L18-L160)