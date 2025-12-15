# Chapter 3: ErgoTree Structure

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Binary representation concepts (bits, bytes, VLQ encoding)
- Basic AST understanding
- Prior chapters: [Chapter 1](./ch01-introduction.md), [Chapter 2](./ch02-type-system.md)

## Learning Objectives

- Parse and interpret ErgoTree header bytes
- Understand constant segregation benefits
- Explain version mechanism and soft-fork support
- Describe complete ErgoTree binary format

## ErgoTree Overview

**ErgoTree** is the serialized, self-sufficient contract representation[^1][^2]:

- Header byte defining format and version
- Optional constants array (if segregation enabled)
- Root expression of type `SigmaProp`
- Optional size field (for v1+)

```zig
const ErgoTree = struct {
    header: HeaderType,
    constants: []const Constant,
    root: union(enum) {
        parsed: SigmaPropValue,
        unparsed: UnparsedTree,
    },
    proposition_bytes: ?[]const u8,

    pub fn bytes(self: *ErgoTree, allocator: Allocator) ![]u8 {
        if (self.proposition_bytes) |b| return b;
        return try serialize(self, allocator);
    }

    pub fn bytesHex(self: *ErgoTree, allocator: Allocator) ![]u8 {
        const b = try self.bytes(allocator);
        return std.fmt.allocPrint(allocator, "{x}", .{b});
    }
};
```

## Header Format

The first byte uses a bit-field format[^3][^4]:

```
   7  6  5  4  3  2  1  0
 ┌──┬──┬──┬──┬──┬──┬──┬──┐
 │  │  │  │  │  │  │  │  │
 └──┴──┴──┴──┴──┴──┴──┴──┘
  │  │  │  │  │  └──┴──┴── Version (bits 0-2)
  │  │  │  │  └─────────── Size flag (bit 3)
  │  │  │  └────────────── Constant segregation (bit 4)
  │  │  └───────────────── Reserved (bit 5, must be 0)
  │  └──────────────────── Reserved for GZIP (bit 6, must be 0)
  └─────────────────────── Extended header (bit 7)
```

```zig
const HeaderType = packed struct(u8) {
    version: u3,              // bits 0-2
    has_size: bool,           // bit 3
    constant_segregation: bool, // bit 4
    reserved1: bool = false,  // bit 5
    reserved_gzip: bool = false, // bit 6
    multi_byte: bool = false, // bit 7

    pub const VERSION_MASK: u8 = 0x07;
    pub const SIZE_FLAG: u8 = 0x08;
    pub const CONST_SEG_FLAG: u8 = 0x10;

    pub fn fromByte(byte: u8) HeaderType {
        return @bitCast(byte);
    }

    pub fn toByte(self: HeaderType) u8 {
        return @bitCast(self);
    }

    pub fn v0(constant_segregation: bool) HeaderType {
        return .{
            .version = 0,
            .has_size = false,
            .constant_segregation = constant_segregation,
        };
    }

    pub fn v1(constant_segregation: bool) HeaderType {
        return .{
            .version = 1,
            .has_size = true, // Required for v1+
            .constant_segregation = constant_segregation,
        };
    }
};
```

### Common Header Values

| Byte | Binary | Meaning |
|------|--------|---------|
| `0x00` | `00000000` | v0, no segregation, no size |
| `0x08` | `00001000` | v0, no segregation, with size |
| `0x10` | `00010000` | v0, constant segregation, no size |
| `0x18` | `00011000` | v0, constant segregation, with size |
| `0x09` | `00001001` | v1, with size (required) |
| `0x19` | `00011001` | v1, constant segregation, with size |

## Binary Format

```
┌───────────────────────────────────────────────────────────────────┐
│                            ErgoTree                               │
├─────────┬─────────────┬──────────────────┬───────────────────────┤
│ Header  │ [Size]      │ [Constants]      │ Root Expression       │
│ 1 byte  │ VLQ (opt)   │ Array (opt)      │ Serialized tree       │
└─────────┴─────────────┴──────────────────┴───────────────────────┘

If header bit 3 is set (hasSize):
  Size = VLQ-encoded size of (Constants + Root Expression)

If header bit 4 is set (isConstantSegregation):
  Constants = VLQ count + Array of serialized constants

Root Expression = Serialized expression tree (SigmaPropValue)
```

```zig
const ErgoTreeSerializer = struct {
    pub fn deserialize(reader: anytype) !ErgoTree {
        // 1. Read header byte
        const header = HeaderType.fromByte(try reader.readByte());

        // 2. Read extended header if bit 7 set
        if (header.multi_byte) {
            // VLQ continuation - read additional bytes
            _ = try readVlqExtension(reader);
        }

        // 3. Read size if flag set
        var tree_size: ?u32 = null;
        if (header.has_size) {
            tree_size = try readVlq(reader);
        }

        // 4. Read constants if segregation enabled
        var constants: []Constant = &.{};
        if (header.constant_segregation) {
            const count = try readVlq(reader);
            constants = try allocator.alloc(Constant, count);
            for (constants) |*c| {
                c.* = try Constant.deserialize(reader);
            }
        }

        // 5. Read root expression
        const root = try Expr.deserialize(reader);

        return ErgoTree{
            .header = header,
            .constants = constants,
            .root = .{ .parsed = root },
            .proposition_bytes = null,
        };
    }
};
```

## Constant Segregation

Extracts constants from expression tree for optimization[^5]:

```
Without segregation:
┌─────────────────────────────────────────────────┐
│ header: 0x00                                    │
│ root: AND(GT(HEIGHT, IntConstant(100)), pk)     │
└─────────────────────────────────────────────────┘

With segregation:
┌─────────────────────────────────────────────────┐
│ header: 0x10                                    │
│ constants: [IntConstant(100)]                   │
│ root: AND(GT(HEIGHT, Placeholder(0)), pk)       │
└─────────────────────────────────────────────────┘
```

Benefits:
1. **Template sharing**: Same template, different constants
2. **Caching**: Templates cached for repeated evaluation
3. **Substitution**: Constants replaced without re-parsing

```zig
/// Substitute ConstantPlaceholder nodes with actual constants
pub fn substConstants(
    root: *const Expr,
    constants: []const Constant,
) Expr {
    return switch (root.*) {
        .constant_placeholder => |ph| .{
            .constant = constants[ph.index],
        },
        .and => |a| .{
            .and = .{
                .left = substConstants(a.left, constants),
                .right = substConstants(a.right, constants),
            },
        },
        // ... other node types
        else => root.*,
    };
}
```

## Version Mechanism

Supports soft-fork compatibility[^6]:

| Version | Block Version | Features |
|---------|---------------|----------|
| v0 | 1 | Original format, AOT costing |
| v1 | 2 | JIT costing, size required |
| v2 | 3 | Extended operations |
| v3 | 4 | UnsignedBigInt, enhanced methods |

```zig
pub fn setVersionBits(header: HeaderType, version: u3) HeaderType {
    var h = header;
    h.version = version;
    // Size flag required for version > 0
    if (version > 0) {
        h.has_size = true;
    }
    return h;
}
```

## Unparsed Trees

When deserialization fails (unknown opcode), raw bytes are stored[^7]:

```zig
const UnparsedTree = struct {
    bytes: []const u8,
    err: DeserializationError,
};

/// Convert to proposition, handling unparsed case
pub fn toProposition(self: *const ErgoTree, replace_constants: bool) !SigmaPropValue {
    return switch (self.root) {
        .parsed => |tree| blk: {
            if (replace_constants and self.constants.len > 0) {
                break :blk substConstants(tree, self.constants);
            }
            break :blk tree;
        },
        .unparsed => |u| return u.err,
    };
}
```

This enables nodes to handle future versions gracefully.

## Creating ErgoTrees

```zig
pub fn fromProposition(prop: SigmaPropValue) ErgoTree {
    return fromPropositionWithHeader(HeaderType.v0(false), prop);
}

pub fn fromPropositionWithHeader(header: HeaderType, prop: SigmaPropValue) ErgoTree {
    // Simple constants don't need segregation
    if (prop == .sigma_prop_constant) {
        return withoutSegregation(header, prop);
    }
    // Complex expressions benefit from segregation
    return withSegregation(header, prop);
}

fn withSegregation(header: HeaderType, prop: SigmaPropValue) ErgoTree {
    var constants = std.ArrayList(Constant).init(allocator);
    const segregated = extractConstants(prop, &constants);
    return ErgoTree{
        .header = .{
            .version = header.version,
            .has_size = header.has_size,
            .constant_segregation = true,
        },
        .constants = constants.toOwnedSlice(),
        .root = .{ .parsed = segregated },
        .proposition_bytes = null,
    };
}
```

## Template Extraction

The **template** is root expression bytes without constant values:

```zig
pub fn template(self: *const ErgoTree) ![]const u8 {
    // Serialize root with placeholders (no constant substitution)
    var buf = std.ArrayList(u8).init(allocator);
    try self.root.parsed.serialize(buf.writer());
    return buf.toOwnedSlice();
}
```

Templates are useful for:
- Identifying script patterns regardless of constants
- Contract template matching
- Caching deserialized templates

## Properties

```zig
const ErgoTree = struct {
    // ... fields ...

    /// Returns true if tree contains deserialization operations
    pub fn hasDeserialize(self: *const ErgoTree) bool {
        return switch (self.root) {
            .parsed => |p| containsDeserializeOp(p),
            .unparsed => false,
        };
    }

    /// Returns true if tree uses blockchain context
    pub fn isUsingBlockchainContext(self: *const ErgoTree) bool {
        return switch (self.root) {
            .parsed => |p| containsContextOp(p),
            .unparsed => false,
        };
    }

    /// Convert to SigmaBoolean if simple proposition
    pub fn toSigmaBooleanOpt(self: *const ErgoTree) ?SigmaBoolean {
        const prop = self.toProposition(self.header.constant_segregation) catch return null;
        return switch (prop) {
            .sigma_prop_constant => |c| c.value,
            else => null,
        };
    }
};
```

## Summary

- ErgoTree is self-sufficient serialized contract format
- **Header byte** encodes version (3 bits), size flag, segregation flag
- **Constant segregation** extracts constants for template sharing
- Version > 0 **requires** size flag for forward compatibility
- **UnparsedTree** handles soft-fork scenarios gracefully
- Trees convert to **SigmaBoolean** for simple cryptographic propositions

---

*Next: [Chapter 4: Value Nodes](../part2/ch04-value-nodes.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/ast/ErgoTree.scala:24-80`

[^2]: Rust: `ergotree-ir/src/ergo_tree.rs:33-41`

[^3]: Scala: `data/shared/src/main/scala/sigma/ast/ErgoTree.scala:227-270`

[^4]: Rust: `ergotree-ir/src/ergo_tree/tree_header.rs:10-32`

[^5]: Scala: `data/shared/src/main/scala/sigma/ast/ErgoTree.scala:307-322`

[^6]: Scala: `data/shared/src/main/scala/sigma/ast/ErgoTree.scala:263-305`

[^7]: Scala: `data/shared/src/main/scala/sigma/ast/ErgoTree.scala:19-22`