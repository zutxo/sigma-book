# Chapter 3: ErgoTree Structure

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Binary representation concepts (bits, bytes, bitwise operations)
- Variable-Length Quantity (VLQ) encoding—a method for encoding integers using a variable number of bytes
- Understanding of Abstract Syntax Trees (ASTs) as hierarchical representations of code structure
- Prior chapters: [Chapter 1](./ch01-introduction.md) for the three-layer architecture, [Chapter 2](./ch02-type-system.md) for type codes used in serialization

## Learning Objectives

By the end of this chapter, you will be able to:

- Parse and interpret ErgoTree header bytes, extracting version and feature flags
- Explain how constant segregation enables template sharing and caching optimizations
- Describe the version mechanism and how it enables soft-fork protocol upgrades
- Read and write the complete ErgoTree binary format

## ErgoTree Overview

When you write an ErgoScript contract, the compiler transforms it into **ErgoTree**—a compact binary format designed for blockchain storage and deterministic execution. Every UTXO box contains an ErgoTree that defines its spending conditions.

ErgoTree is specifically designed to be[^1][^2]:
- **Self-sufficient**: Contains everything needed for evaluation (no external dependencies)
- **Compact**: Optimized binary encoding minimizes on-chain storage
- **Forward-compatible**: Version mechanism enables protocol upgrades without hard forks
- **Deterministic**: Same bytes always produce the same evaluation result

The structure consists of:
- **Header byte** — Format version and feature flags
- **Size field** (optional) — Total size for fast skipping
- **Constants array** (optional) — Extracted constants for template sharing
- **Root expression** — The actual script logic, returning `SigmaProp`

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
┌──────────────────────────────────────────────────────────────────┐
│                            ErgoTree                              │
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
            // Bounds check: prevent DoS via excessive allocation
            const MAX_CONSTANTS: u32 = 4096;
            if (count > MAX_CONSTANTS) {
                return error.TooManyConstants;
            }
            constants = try allocator.alloc(Constant, count);
            for (constants) |*c| {
                c.* = try Constant.deserialize(reader);
            }
        }
        // NOTE: In production, use a pre-allocated pool instead of dynamic
        // allocation during deserialization. See ZIGMA_STYLE.md.

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

Constant segregation is an optimization technique that extracts literal values from the expression tree and stores them in a separate array[^5]. The expression tree then references these constants via placeholder indices. This seemingly simple change enables several powerful optimizations:

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
// NOTE: In production, use an iterative approach with an explicit work stack
// to guarantee bounded stack depth and prevent stack overflow on deep trees.
```

## Version Mechanism

The ErgoTree version field (bits 0-2) enables soft-fork protocol upgrades without breaking consensus[^6]. Each ErgoTree version corresponds to a minimum required block version in the Ergo protocol—nodes running older protocol versions will skip validation of scripts with newer ErgoTree versions rather than rejecting them as invalid.

| ErgoTree Version | Min Block Version | Key Features |
|------------------|-------------------|--------------|
| v0 | 1 | Original format with Ahead-of-Time (AOT) costing calculated during compilation |
| v1 | 2 | Just-in-Time (JIT) costing calculated during execution; size field required |
| v2 | 3 | Extended operations and new opcodes |
| v3 | 4 | `UnsignedBigInt` type and enhanced collection methods |

The size field became mandatory in v1 to support forward compatibility—nodes can skip over scripts they cannot fully parse by reading the size and advancing past the unknown content.

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

When a node encounters an ErgoTree with an unknown opcode—typically from a newer protocol version—deserialization fails. Rather than rejecting the transaction entirely, the raw bytes are preserved as an "unparsed tree"[^7]. This design is critical for soft-fork compatibility: older nodes can process blocks containing newer script versions without understanding their contents.

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

This chapter covered the complete ErgoTree binary format—the serialized representation of smart contracts stored in every UTXO box:

- **ErgoTree** is a self-sufficient serialized contract format containing everything needed for evaluation without external dependencies
- The **header byte** uses a bit-field layout: version (bits 0-2), size flag (bit 3), constant segregation flag (bit 4), with reserved bits for future extensions
- **Constant segregation** (bit 4) extracts literal values into a separate array, enabling template sharing, caching, and runtime substitution without re-parsing
- The **version mechanism** enables soft-fork protocol upgrades—newer ErgoTree versions are skipped by older nodes rather than causing consensus failures
- ErgoTree versions 1+ **require** the size flag, allowing nodes to skip past unknown content
- **UnparsedTree** preserves raw bytes when deserialization fails, maintaining block validity even with unknown opcodes
- Simple cryptographic propositions can be extracted as **SigmaBoolean** values for direct signature verification

---

*Next: [Chapter 4: Value Nodes](../part2/ch04-value-nodes.md)*

[^1]: Scala: [`ErgoTree.scala:24-80`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/ErgoTree.scala#L24-L80)

[^2]: Rust: [`ergo_tree.rs:33-41`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/ergo_tree.rs#L33-L41)

[^3]: Scala: [`ErgoTree.scala:227-270`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/ErgoTree.scala#L227-L270)

[^4]: Rust: [`tree_header.rs:10-32`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/ergo_tree/tree_header.rs#L10-L32)

[^5]: Scala: [`ErgoTree.scala:307-322`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/ErgoTree.scala#L307-L322)

[^6]: Scala: [`ErgoTree.scala:263-305`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/ErgoTree.scala#L263-L305)

[^7]: Scala: [`ErgoTree.scala:19-22`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/ErgoTree.scala#L19-L22)
