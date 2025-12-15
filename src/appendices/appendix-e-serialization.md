# Appendix E: Serialization Format Reference

Complete reference for ErgoTree and value serialization formats[^1][^2].

## Integer Encoding

### VLQ (Variable-Length Quantity)

```
VLQ Encoding
══════════════════════════════════════════════════════════════════

Byte format: [C][D D D D D D D]
             |  |____________|
             |       |
             |       +-- 7 data bits
             +---------- Continuation bit (1 = more bytes follow)

Examples:
  0       → [0x00]                    (1 byte)
  127     → [0x7F]                    (1 byte)
  128     → [0x80, 0x01]              (2 bytes: 10000000 00000001)
  16383   → [0xFF, 0x7F]              (2 bytes)
  16384   → [0x80, 0x80, 0x01]        (3 bytes)
```

```zig
const VlqEncoder = struct {
    /// Encode unsigned integer as VLQ
    pub fn encodeU64(value: u64, writer: anytype) !void {
        var v = value;
        while (v >= 0x80) {
            try writer.writeByte(@as(u8, @truncate(v)) | 0x80);
            v >>= 7;
        }
        try writer.writeByte(@as(u8, @truncate(v)));
    }

    /// Decode VLQ to unsigned integer
    pub fn decodeU64(reader: anytype) !u64 {
        var result: u64 = 0;
        var shift: u6 = 0;
        while (true) {
            const byte = try reader.readByte();
            result |= @as(u64, byte & 0x7F) << shift;
            if (byte & 0x80 == 0) break;
            shift += 7;
            if (shift > 63) return error.VlqOverflow;
        }
        return result;
    }
};
```

### ZigZag Encoding

```zig
const ZigZag = struct {
    /// Encode signed → unsigned (small negatives stay small)
    pub fn encode32(n: i32) u32 {
        return @bitCast((n << 1) ^ (n >> 31));
    }

    pub fn encode64(n: i64) u64 {
        return @bitCast((n << 1) ^ (n >> 63));
    }

    /// Decode unsigned → signed
    pub fn decode32(n: u32) i32 {
        return @bitCast((n >> 1) ^ (~(n & 1) +% 1));
    }

    pub fn decode64(n: u64) i64 {
        return @bitCast((n >> 1) ^ (~(n & 1) +% 1));
    }
};

// Mapping: 0 → 0, -1 → 1, 1 → 2, -2 → 3, 2 → 4, ...
```

## Type Serialization[^3]

### Primitive Type Codes

| Type | Dec | Hex | Zig |
|------|-----|-----|-----|
| SBoolean | 1 | 0x01 | `.boolean` |
| SByte | 2 | 0x02 | `.byte` |
| SShort | 3 | 0x03 | `.short` |
| SInt | 4 | 0x04 | `.int` |
| SLong | 5 | 0x05 | `.long` |
| SBigInt | 6 | 0x06 | `.big_int` |
| SGroupElement | 7 | 0x07 | `.group_element` |
| SSigmaProp | 8 | 0x08 | `.sigma_prop` |
| SUnsignedBigInt | 9 | 0x09 | `.unsigned_big_int` |

### Collection Types

```zig
const TypeEncoder = struct {
    const COLL_BASE: u8 = 12;      // 0x0C
    const NESTED_COLL: u8 = 24;    // 0x18
    const OPTION_BASE: u8 = 36;    // 0x24

    /// Encode collection type
    pub fn encodeColl(elem: SType) u8 {
        if (elem.isPrimitive()) {
            return COLL_BASE + elem.typeCode();
        }
        if (elem == .coll) {
            return NESTED_COLL + elem.inner().typeCode();
        }
        // Non-embeddable: write COLL_BASE then element type separately
        return COLL_BASE;
    }
};
```

### Non-Embeddable Types

| Type | Dec | Hex |
|------|-----|-----|
| SBox | 99 | 0x63 |
| SAvlTree | 100 | 0x64 |
| SContext | 101 | 0x65 |
| SHeader | 104 | 0x68 |
| SPreHeader | 105 | 0x69 |
| SGlobal | 106 | 0x6A |

## ErgoTree Format[^4]

### Header Byte

```
ErgoTree Header
══════════════════════════════════════════════════════════════════

Bits: [V V V V][S][C][R][R]
      |______|  |  |  |__|
         |      |  |    |
         |      |  |    +-- Reserved (2 bits)
         |      |  +------- Constant segregation (1 = segregated)
         |      +---------- Size flag (1 = size bytes present)
         +----------------- Version (4 bits, 0-15)

Version Mapping:
  0 → ErgoTree v0 (protocol v3.x)
  1 → ErgoTree v1 (protocol v4.x)
  2 → ErgoTree v2 (protocol v5.x, JIT costing)
  3 → ErgoTree v3 (protocol v6.x)
```

```zig
const ErgoTreeHeader = struct {
    version: u4,
    has_size: bool,
    constant_segregation: bool,

    pub fn parse(byte: u8) ErgoTreeHeader {
        return .{
            .version = @truncate(byte >> 4),
            .has_size = (byte & 0x08) != 0,
            .constant_segregation = (byte & 0x04) != 0,
        };
    }

    pub fn serialize(self: ErgoTreeHeader) u8 {
        var result: u8 = @as(u8, self.version) << 4;
        if (self.has_size) result |= 0x08;
        if (self.constant_segregation) result |= 0x04;
        return result;
    }
};
```

### Complete Structure

```
ErgoTree Wire Format
══════════════════════════════════════════════════════════════════

┌─────────┬──────────┬──────────────┬─────────────┬──────────────┐
│ Header  │   Size   │  Constants   │ Complexity  │    Root      │
│ 1 byte  │   VLQ    │    Array     │    VLQ      │ Expression   │
│         │(optional)│  (if C=1)    │ (optional)  │              │
└─────────┴──────────┴──────────────┴─────────────┴──────────────┘

With constant segregation (C=1):
┌─────────┬──────────┬───────────┬──────────────────────────────┐
│ Header  │ # consts │ Constants │   Root (with placeholders)   │
│         │   VLQ    │  [type +  │                              │
│         │          │  value]*  │                              │
└─────────┴──────────┴───────────┴──────────────────────────────┘
```

## Value Serialization[^5]

### Primitive Values

```zig
const DataSerializer = struct {
    pub fn serialize(value: Value, writer: anytype) !void {
        switch (value) {
            .boolean => |b| try writer.writeByte(if (b) 0x01 else 0x00),
            .byte => |b| try writer.writeByte(@bitCast(b)),
            .short => |s| try VlqEncoder.encodeI16(s, writer),
            .int => |i| try VlqEncoder.encodeI32(i, writer),
            .long => |l| try VlqEncoder.encodeI64(l, writer),
            .big_int => |bi| try serializeBigInt(bi, writer),
            .group_element => |ge| try ge.serializeCompressed(writer),
            .sigma_prop => |sp| try serializeSigmaProp(sp, writer),
            .coll => |c| try serializeColl(c, writer),
            // ...
        }
    }

    fn serializeBigInt(bi: BigInt256, writer: anytype) !void {
        const bytes = bi.toBytesBigEndian();
        // Skip leading zeros for signed representation
        var start: usize = 0;
        while (start < bytes.len - 1 and bytes[start] == 0) : (start += 1) {}
        try writer.writeByte(@intCast(bytes.len - start));
        try writer.writeAll(bytes[start..]);
    }
};
```

### GroupElement (SEC1 Compressed)

```
GroupElement Encoding (33 bytes)
══════════════════════════════════════════════════════════════════

┌────────────┬─────────────────────────────────────────────────────┐
│   Prefix   │                 X Coordinate                        │
│  (1 byte)  │                  (32 bytes)                         │
├────────────┼─────────────────────────────────────────────────────┤
│ 0x02 = Y   │                                                     │
│    even    │              Big-endian X value                     │
│ 0x03 = Y   │                                                     │
│    odd     │                                                     │
└────────────┴─────────────────────────────────────────────────────┘
```

### SigmaProp

```zig
const SigmaPropSerializer = struct {
    const PROVE_DLOG: u8 = 0xCD;
    const PROVE_DHT: u8 = 0xCE;
    const THRESHOLD: u8 = 0x98;
    const AND: u8 = 0x96;
    const OR: u8 = 0x97;

    pub fn serialize(sp: SigmaBoolean, writer: anytype) !void {
        switch (sp) {
            .prove_dlog => |pk| {
                try writer.writeByte(PROVE_DLOG);
                try pk.serializeCompressed(writer);
            },
            .prove_dht => |dht| {
                try writer.writeByte(PROVE_DHT);
                try dht.g.serializeCompressed(writer);
                try dht.h.serializeCompressed(writer);
                try dht.u.serializeCompressed(writer);
                try dht.v.serializeCompressed(writer);
            },
            .and_conj => |children| {
                try writer.writeByte(AND);
                try VlqEncoder.encodeU64(children.len, writer);
                for (children) |child| try serialize(child, writer);
            },
            .or_conj => |children| {
                try writer.writeByte(OR);
                try VlqEncoder.encodeU64(children.len, writer);
                for (children) |child| try serialize(child, writer);
            },
            .threshold => |t| {
                try writer.writeByte(THRESHOLD);
                try VlqEncoder.encodeU64(t.k, writer);
                try VlqEncoder.encodeU64(t.children.len, writer);
                for (t.children) |child| try serialize(child, writer);
            },
        }
    }
};
```

### Collections

```zig
const CollSerializer = struct {
    pub fn serialize(coll: Collection, writer: anytype) !void {
        try VlqEncoder.encodeU64(coll.len, writer);
        // Element type already encoded in type header
        for (coll.items) |item| {
            try DataSerializer.serialize(item, writer);
        }
    }

    /// Optimized boolean collection (bit-packed)
    pub fn serializeBoolColl(bools: []const bool, writer: anytype) !void {
        try VlqEncoder.encodeU64(bools.len, writer);
        var byte: u8 = 0;
        var bit: u3 = 0;
        for (bools) |b| {
            if (b) byte |= @as(u8, 1) << bit;
            bit +%= 1;
            if (bit == 0) {
                try writer.writeByte(byte);
                byte = 0;
            }
        }
        if (bools.len % 8 != 0) try writer.writeByte(byte);
    }
};
```

## Expression Serialization[^6]

### General Pattern

```zig
const ExprSerializer = struct {
    pub fn serialize(expr: Expr, writer: anytype) !void {
        // Write opcode
        try writer.writeByte(@intFromEnum(expr.opCode()));

        // Write opcode-specific data
        switch (expr) {
            .val_use => |vu| try VlqEncoder.encodeU32(vu.id, writer),
            .constant_placeholder => |cp| {
                try VlqEncoder.encodeU32(cp.index, writer);
                try TypeEncoder.serialize(cp.tpe, writer);
            },
            .bin_op => |bo| {
                try serialize(bo.left.*, writer);
                try serialize(bo.right.*, writer);
            },
            .method_call => |mc| {
                try writer.writeByte(mc.type_code);
                try writer.writeByte(mc.method_id);
                try serialize(mc.receiver.*, writer);
                try VlqEncoder.encodeU64(mc.args.len, writer);
                for (mc.args) |arg| try serialize(arg.*, writer);
            },
            // ...
        }
    }
};
```

### Block Expressions

```
Block Value Structure
══════════════════════════════════════════════════════════════════

BlockValue:
┌────────┬──────────┬─────────────────────┬───────────────────────┐
│ 0xD8   │  count   │   ValDef items      │   Result expr         │
│        │   VLQ    │                     │                       │
└────────┴──────────┴─────────────────────┴───────────────────────┘

ValDef:
┌────────┬────────┬────────────┬───────────────────────────────────┐
│ 0xD6   │   ID   │   Type     │        RHS Expression             │
│        │  VLQ   │ (optional) │                                   │
└────────┴────────┴────────────┴───────────────────────────────────┘

FuncValue (Lambda):
┌────────┬──────────┬─────────────────────┬───────────────────────┐
│ 0xD9   │ arg cnt  │  Args (ID + type)   │   Body expr           │
│        │   VLQ    │                     │                       │
└────────┴──────────┴─────────────────────┴───────────────────────┘
```

## Size Limits[^7]

| Limit | Value | Description |
|-------|-------|-------------|
| Max ErgoTree size | 4 KB | Serialized bytes |
| Max box size | 4 KB | Total serialized |
| Max constants | 255 | Per ErgoTree |
| Max registers | 10 | R0-R9 |
| Max tokens/box | 255 | Token types |
| Max BigInt bytes | 32 | 256 bits |

## Deserialization

```zig
const SigmaByteReader = struct {
    reader: std.io.Reader,
    constant_store: []const Constant,
    version: ErgoTreeVersion,

    pub fn readVlqU64(self: *SigmaByteReader) !u64 {
        return VlqEncoder.decodeU64(self.reader);
    }

    pub fn readType(self: *SigmaByteReader) !SType {
        const code = try self.reader.readByte();
        return TypeEncoder.decode(code, self);
    }

    pub fn readExpr(self: *SigmaByteReader) !Expr {
        const opcode = try self.reader.readByte();
        if (opcode <= 0x70) {
            // Constant (type code in data region)
            return try self.readConstantWithType(opcode);
        }
        return try ExprSerializer.deserialize(@enumFromInt(opcode), self);
    }
};
```

---

*[Previous: Appendix D](./appendix-d-methods.md) | [Next: Appendix F](./appendix-f-version-history.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/serialization/`

[^2]: Rust: `ergotree-ir/src/serialization/sigma_byte_reader.rs:12-97`

[^3]: Rust: `ergotree-ir/src/serialization/types.rs`

[^4]: Rust: `ergotree-ir/src/ergo_tree.rs` (header parsing)

[^5]: Rust: `ergotree-ir/src/serialization/data.rs`

[^6]: Rust: `ergotree-ir/src/serialization/expr.rs`

[^7]: Scala: `docs/spec/serialization.tex` (size limits)
