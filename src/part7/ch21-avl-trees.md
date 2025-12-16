# Chapter 21: AVL+ Trees

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Hash functions ([Chapter 10](../part4/ch10-hash-functions.md))
- Collections ([Chapter 20](./ch20-collections.md))
- Binary search tree concepts

## Learning Objectives

- Understand AVL+ trees as authenticated dictionaries
- Implement digest structure and operation flags
- Master proof-based verification model
- Work with tree operations (contains, get, insert, update, remove)
- Know cost implications of authenticated operations

## Authenticated Dictionary Model

AVL+ trees provide authenticated key-value storage[^1][^2]:

```
Prover-Verifier Architecture
─────────────────────────────────────────────────────

OFF-CHAIN (Prover - holds full tree):
┌─────────────────────────────────────────────────────┐
│                 BatchAVLProver                      │
│  ┌─────────────────────────────────────────────────┐│
│  │           Complete Tree Structure               ││
│  │                    [H]                          ││
│  │                   /   \                         ││
│  │                 [D]   [L]                       ││
│  │                /   \ /   \                      ││
│  │              [B] [F][J] [N]                     ││
│  └─────────────────────────────────────────────────┘│
│  • Performs operations                              │
│  • Generates proofs                                 │
│  • Maintains full state                             │
└─────────────────────────│───────────────────────────┘
                          │ proof bytes
                          ▼
ON-CHAIN (Verifier - holds only digest):
┌─────────────────────────────────────────────────────┐
│               CAvlTreeVerifier                      │
│  ┌─────────────────────────────────────────────────┐│
│  │  Digest: [32-byte root hash][height byte]       ││
│  │          ═══════════════════════════════        ││
│  │          (33 bytes total)                       ││
│  └─────────────────────────────────────────────────┘│
│  • Verifies proof bytes                             │
│  • Returns operation results                        │
│  • Rejects invalid proofs                           │
└─────────────────────────────────────────────────────┘
```

## AvlTreeData Structure

Core data type for authenticated trees[^3][^4]:

```zig
/// Authenticated tree data (stored on-chain)
const AvlTreeData = struct {
    /// Root hash + height (33 bytes total)
    digest: ADDigest,
    /// Permitted operations
    tree_flags: AvlTreeFlags,
    /// Fixed key length (all keys same size)
    key_length: u32,
    /// Optional fixed value length
    value_length_opt: ?u32,

    pub const DIGEST_SIZE: usize = 33; // 32-byte hash + 1-byte height

    pub fn fromDigest(digest: []const u8) AvlTreeData {
        return .{
            .digest = ADDigest.fromSlice(digest),
            .tree_flags = AvlTreeFlags.allOperationsAllowed(),
            .key_length = 32, // Default: 32-byte keys
            .value_length_opt = null,
        };
    }
};

/// 33-byte authenticated digest
const ADDigest = struct {
    /// 32-byte BLAKE2b256 root hash
    root_hash: [32]u8,
    /// Tree height (0-255)
    height: u8,

    pub fn fromSlice(bytes: []const u8) ADDigest {
        var result: ADDigest = undefined;
        @memcpy(&result.root_hash, bytes[0..32]);
        result.height = bytes[32];
        return result;
    }

    pub fn toBytes(self: ADDigest) [33]u8 {
        var result: [33]u8 = undefined;
        @memcpy(result[0..32], &self.root_hash);
        result[32] = self.height;
        return result;
    }
};
```

## Operation Flags

Control which modifications are permitted[^5][^6]:

```zig
/// Operation permission flags (bit-packed)
const AvlTreeFlags = struct {
    flags: u8,

    /// Bit positions
    const INSERT_BIT: u8 = 0x01;
    const UPDATE_BIT: u8 = 0x02;
    const REMOVE_BIT: u8 = 0x04;

    pub fn new(insert_allowed: bool, update_allowed: bool, remove_allowed: bool) AvlTreeFlags {
        var flags: u8 = 0;
        if (insert_allowed) flags |= INSERT_BIT;
        if (update_allowed) flags |= UPDATE_BIT;
        if (remove_allowed) flags |= REMOVE_BIT;
        return .{ .flags = flags };
    }

    pub fn parse(byte: u8) AvlTreeFlags {
        return .{ .flags = byte };
    }

    pub fn serialize(self: AvlTreeFlags) u8 {
        return self.flags;
    }

    // Predefined flag combinations
    pub fn readOnly() AvlTreeFlags {
        return .{ .flags = 0x00 };
    }

    pub fn allOperationsAllowed() AvlTreeFlags {
        return .{ .flags = 0x07 };
    }

    pub fn insertOnly() AvlTreeFlags {
        return .{ .flags = 0x01 };
    }

    pub fn removeOnly() AvlTreeFlags {
        return .{ .flags = 0x04 };
    }

    // Permission checks
    pub fn insertAllowed(self: AvlTreeFlags) bool {
        return (self.flags & INSERT_BIT) != 0;
    }

    pub fn updateAllowed(self: AvlTreeFlags) bool {
        return (self.flags & UPDATE_BIT) != 0;
    }

    pub fn removeAllowed(self: AvlTreeFlags) bool {
        return (self.flags & REMOVE_BIT) != 0;
    }
};
```

## AvlTree Interface

ErgoScript interface for authenticated trees[^7]:

```zig
/// AvlTree wrapper providing ErgoScript interface
const AvlTree = struct {
    data: AvlTreeData,

    /// Get 33-byte authenticated digest
    pub fn digest(self: *const AvlTree) []const u8 {
        return &self.data.digest.toBytes();
    }

    /// Get operation flags byte
    pub fn enabledOperations(self: *const AvlTree) u8 {
        return self.data.tree_flags.serialize();
    }

    /// Get fixed key length
    pub fn keyLength(self: *const AvlTree) i32 {
        return @intCast(self.data.key_length);
    }

    /// Get optional fixed value length
    pub fn valueLengthOpt(self: *const AvlTree) ?i32 {
        if (self.data.value_length_opt) |v| {
            return @intCast(v);
        }
        return null;
    }

    /// Permission checks
    pub fn isInsertAllowed(self: *const AvlTree) bool {
        return self.data.tree_flags.insertAllowed();
    }

    pub fn isUpdateAllowed(self: *const AvlTree) bool {
        return self.data.tree_flags.updateAllowed();
    }

    pub fn isRemoveAllowed(self: *const AvlTree) bool {
        return self.data.tree_flags.removeAllowed();
    }

    /// Create new tree with updated digest (immutable)
    pub fn updateDigest(self: *const AvlTree, new_digest: []const u8) AvlTree {
        var new_data = self.data;
        new_data.digest = ADDigest.fromSlice(new_digest);
        return .{ .data = new_data };
    }

    /// Create new tree with updated flags (immutable)
    pub fn updateOperations(self: *const AvlTree, new_flags: u8) AvlTree {
        var new_data = self.data;
        new_data.tree_flags = AvlTreeFlags.parse(new_flags);
        return .{ .data = new_data };
    }
};
```

## Verifier Implementation

The verifier processes proofs to verify operations[^8][^9]:

```zig
/// AVL tree proof verifier
const AvlTreeVerifier = struct {
    /// Current state digest (None if verification failed)
    current_digest: ?ADDigest,
    /// Proof bytes to process
    proof: []const u8,
    /// Current position in proof
    proof_pos: usize,
    /// Key length
    key_length: usize,
    /// Optional value length
    value_length_opt: ?usize,

    pub fn init(tree: *const AvlTree, proof: []const u8) AvlTreeVerifier {
        return .{
            .current_digest = tree.data.digest,
            .proof = proof,
            .proof_pos = 0,
            .key_length = tree.data.key_length,
            .value_length_opt = tree.data.value_length_opt,
        };
    }

    /// Get current tree height
    pub fn treeHeight(self: *const AvlTreeVerifier) usize {
        if (self.current_digest) |d| {
            return d.height;
        }
        return 0;
    }

    /// Get current digest (None if verification failed)
    pub fn digest(self: *const AvlTreeVerifier) ?[]const u8 {
        if (self.current_digest) |d| {
            return &d.toBytes();
        }
        return null;
    }

    /// Perform lookup operation
    pub fn performLookup(self: *AvlTreeVerifier, key: []const u8) !?[]const u8 {
        if (self.current_digest == null) return error.VerificationFailed;

        // Process proof to verify key existence
        const result = try self.verifyLookup(key);
        return result;
    }

    /// Perform insert operation
    pub fn performInsert(
        self: *AvlTreeVerifier,
        key: []const u8,
        value: []const u8,
    ) !?[]const u8 {
        if (self.current_digest == null) return error.VerificationFailed;

        // Process proof to verify insertion
        const old_value = try self.verifyInsert(key, value);

        // Update digest based on proof
        self.updateDigestFromProof();

        return old_value;
    }

    /// Perform update operation
    pub fn performUpdate(
        self: *AvlTreeVerifier,
        key: []const u8,
        value: []const u8,
    ) !?[]const u8 {
        if (self.current_digest == null) return error.VerificationFailed;

        const old_value = try self.verifyUpdate(key, value);
        self.updateDigestFromProof();
        return old_value;
    }

    /// Perform remove operation
    pub fn performRemove(self: *AvlTreeVerifier, key: []const u8) !?[]const u8 {
        if (self.current_digest == null) return error.VerificationFailed;

        const old_value = try self.verifyRemove(key);
        self.updateDigestFromProof();
        return old_value;
    }

    fn verifyLookup(self: *AvlTreeVerifier, key: []const u8) !?[]const u8 {
        // NOTE: Stub - full implementation requires:
        // 1. Read node type from proof (leaf vs internal)
        // 2. Compare key with node key
        // 3. Follow proof path based on comparison result
        // 4. Verify all hashes match computed values
        // See scorex-util: BatchAVLVerifier for reference.
        _ = self;
        _ = key;
        @compileError("verifyLookup not implemented - see reference implementations");
    }
    // SECURITY: Key comparisons in production must be constant-time to prevent
    // timing attacks that could leak key values. Use std.crypto.utils.timingSafeEql.

    fn updateDigestFromProof(self: *AvlTreeVerifier) void {
        // Extract new digest from proof processing
        _ = self;
    }
};
```

## Proof-Based Operations

Operations use proofs for verification[^10][^11]:

```zig
/// Evaluate contains operation
fn containsEval(
    tree: *const AvlTree,
    key: []const u8,
    proof: []const u8,
    E: *Evaluator,
) !bool {
    // Cost: create verifier O(proof.length)
    try E.addSeqCost(CreateVerifierCost, proof.len, OpCode.AvlTreeContains);
    var verifier = AvlTreeVerifier.init(tree, proof);

    // Cost: lookup O(tree.height)
    const n_items = verifier.treeHeight();
    try E.addSeqCost(LookupCost, n_items, OpCode.AvlTreeContains);

    const result = verifier.performLookup(key) catch return false;
    return result != null;
}

/// Evaluate get operation
fn getEval(
    tree: *const AvlTree,
    key: []const u8,
    proof: []const u8,
    E: *Evaluator,
) !?[]const u8 {
    try E.addSeqCost(CreateVerifierCost, proof.len, OpCode.AvlTreeGet);
    var verifier = AvlTreeVerifier.init(tree, proof);

    const n_items = verifier.treeHeight();
    try E.addSeqCost(LookupCost, n_items, OpCode.AvlTreeGet);

    return verifier.performLookup(key) catch return error.InvalidProof;
}

/// Evaluate insert operation
fn insertEval(
    tree: *const AvlTree,
    entries: []const KeyValue,
    proof: []const u8,
    E: *Evaluator,
) !?*AvlTree {
    // Check permission
    try E.addCost(IsInsertAllowedCost, OpCode.AvlTreeInsert);
    if (!tree.isInsertAllowed()) {
        return null;
    }

    try E.addSeqCost(CreateVerifierCost, proof.len, OpCode.AvlTreeInsert);
    var verifier = AvlTreeVerifier.init(tree, proof);

    const n_items = @max(verifier.treeHeight(), 1);

    // Process each entry
    for (entries) |entry| {
        try E.addSeqCost(InsertCost, n_items, OpCode.AvlTreeInsert);
        _ = verifier.performInsert(entry.key, entry.value) catch return null;
    }

    // Return new tree with updated digest
    const new_digest = verifier.digest() orelse return null;
    try E.addCost(UpdateDigestCost, OpCode.AvlTreeInsert);
    const new_tree = tree.updateDigest(new_digest);
    return &new_tree;
}

/// Evaluate remove operation
fn removeEval(
    tree: *const AvlTree,
    keys: []const []const u8,
    proof: []const u8,
    E: *Evaluator,
) !?*AvlTree {
    try E.addCost(IsRemoveAllowedCost, OpCode.AvlTreeRemove);
    if (!tree.isRemoveAllowed()) {
        return null;
    }

    try E.addSeqCost(CreateVerifierCost, proof.len, OpCode.AvlTreeRemove);
    var verifier = AvlTreeVerifier.init(tree, proof);

    const n_items = @max(verifier.treeHeight(), 1);

    for (keys) |key| {
        try E.addSeqCost(RemoveCost, n_items, OpCode.AvlTreeRemove);
        _ = verifier.performRemove(key) catch return null;
    }

    const new_digest = verifier.digest() orelse return null;
    try E.addCost(UpdateDigestCost, OpCode.AvlTreeRemove);
    return &tree.updateDigest(new_digest);
}

const KeyValue = struct {
    key: []const u8,
    value: []const u8,
};
```

## Cost Model

AVL tree operations have two-part costs[^12]:

```
AVL Tree Operation Costs
─────────────────────────────────────────────────────

Phase 1 - Create Verifier (O(proof.length)):
  base = 110, per_chunk = 20, chunk_size = 64

Phase 2 - Per Operation (O(tree.height)):
Operation     │ Base │ Per Height │ Chunk
──────────────┼──────┼────────────┼───────
Lookup        │  40  │    10      │   1
Insert        │  40  │    10      │   1
Update        │ 120  │    20      │   1
Remove        │ 100  │    15      │   1
──────────────────────────────────────────────────────

Example: Get operation on tree with height 10, proof 128 bytes
  Verifier: 110 + ⌈128/64⌉ × 20 = 110 + 2 × 20 = 150
  Lookup:   40 + 10 × 10 = 140
  Total:    290 JitCost units
```

```zig
const CreateVerifierCost = PerItemCost{
    .base = JitCost{ .value = 110 },
    .per_chunk = JitCost{ .value = 20 },
    .chunk_size = 64,
};

const LookupCost = PerItemCost{
    .base = JitCost{ .value = 40 },
    .per_chunk = JitCost{ .value = 10 },
    .chunk_size = 1,
};

const InsertCost = PerItemCost{
    .base = JitCost{ .value = 40 },
    .per_chunk = JitCost{ .value = 10 },
    .chunk_size = 1,
};

const UpdateCost = PerItemCost{
    .base = JitCost{ .value = 120 },
    .per_chunk = JitCost{ .value = 20 },
    .chunk_size = 1,
};

const RemoveCost = PerItemCost{
    .base = JitCost{ .value = 100 },
    .per_chunk = JitCost{ .value = 15 },
    .chunk_size = 1,
};
```

## Serialization

AvlTreeData serialization format[^13][^14]:

```zig
/// Serialize AvlTreeData
fn serializeAvlTreeData(data: *const AvlTreeData, writer: anytype) !void {
    // Digest (33 bytes)
    try writer.writeAll(&data.digest.toBytes());

    // Flags (1 byte)
    try writer.writeByte(data.tree_flags.serialize());

    // Key length (VLQ)
    try writeUInt(writer, data.key_length);

    // Optional value length
    if (data.value_length_opt) |vlen| {
        try writer.writeByte(1); // Some
        try writeUInt(writer, vlen);
    } else {
        try writer.writeByte(0); // None
    }
}

/// Parse AvlTreeData
fn parseAvlTreeData(reader: anytype) !AvlTreeData {
    // Digest (33 bytes)
    var digest_bytes: [33]u8 = undefined;
    _ = try reader.readAll(&digest_bytes);
    const digest = ADDigest.fromSlice(&digest_bytes);

    // Flags (1 byte)
    const flags = AvlTreeFlags.parse(try reader.readByte());

    // Key length (VLQ)
    const key_length = try readUInt(reader);

    // Optional value length
    const has_value_length = (try reader.readByte()) != 0;
    const value_length_opt: ?u32 = if (has_value_length)
        try readUInt(reader)
    else
        null;

    return AvlTreeData{
        .digest = digest,
        .tree_flags = flags,
        .key_length = key_length,
        .value_length_opt = value_length_opt,
    };
}
```

## Off-Chain Proof Generation

Provers generate proofs for operations:

```zig
/// Off-chain AVL tree prover (holds full tree)
const AvlProver = struct {
    /// Full tree structure
    root: ?*AvlNode,
    /// Key length
    key_length: usize,
    /// Value length (optional)
    value_length_opt: ?usize,
    /// Pending operations for batch proof
    pending_ops: std.ArrayList(Operation),
    allocator: Allocator,

    const Operation = union(enum) {
        lookup: []const u8,
        insert: struct { key: []const u8, value: []const u8 },
        update: struct { key: []const u8, value: []const u8 },
        remove: []const u8,
    };

    /// Perform insert and record for proof
    pub fn performInsert(self: *AvlProver, key: []const u8, value: []const u8) !void {
        // Actually insert into tree
        self.root = try self.insertNode(self.root, key, value);
        // Record for proof generation
        try self.pending_ops.append(.{ .insert = .{ .key = key, .value = value } });
    }

    /// Generate proof for all pending operations
    pub fn generateProof(self: *AvlProver) ![]const u8 {
        var proof_builder = ProofBuilder.init(self.allocator);

        for (self.pending_ops.items) |op| {
            switch (op) {
                .lookup => |key| try proof_builder.addLookupPath(self.root, key),
                .insert => |ins| try proof_builder.addInsertPath(self.root, ins.key),
                .update => |upd| try proof_builder.addUpdatePath(self.root, upd.key),
                .remove => |key| try proof_builder.addRemovePath(self.root, key),
            }
        }

        self.pending_ops.clearRetainingCapacity();
        return proof_builder.finish();
    }

    /// Get current tree digest
    pub fn digest(self: *const AvlProver) ADDigest {
        if (self.root) |r| {
            return computeNodeDigest(r);
        }
        return ADDigest{ .root_hash = [_]u8{0} ** 32, .height = 0 };
    }

    fn computeNodeDigest(node: *const AvlNode) ADDigest {
        _ = node;
        // Compute BLAKE2b256 hash of node contents
        // Include left and right child hashes
        return undefined;
    }
};

const AvlNode = struct {
    key: []const u8,
    value: []const u8,
    left: ?*AvlNode,
    right: ?*AvlNode,
    height: u8,
};
```

## Key Ordering Requirement

Keys must be provided in the same order during proof generation and verification[^15]:

```
CRITICAL: Key Ordering
─────────────────────────────────────────────────────

Proof Generation (off-chain):
  prover.performLookup(key_A)
  prover.performLookup(key_B)
  prover.performLookup(key_C)
  proof = prover.generateProof()

Verification (on-chain):
  tree.getMany([key_A, key_B, key_C], proof)  ✓ Works
  tree.getMany([key_B, key_A, key_C], proof)  ✗ Fails

The proof encodes a specific traversal path.
Different key order = different path = verification failure.
```

## Summary

- **Authenticated dictionaries** store only 33-byte digest on-chain
- **Prover** (off-chain) holds full tree, generates proofs
- **Verifier** (on-chain) verifies proofs with only digest
- **Operation flags** control insert/update/remove permissions
- **Key ordering** must match between proof generation and verification
- **Cost scales** with proof length (verifier creation) and tree height (operations)
- All methods are **immutable**—return new tree instances

---

*Next: [Chapter 22: Box Model](./ch22-box-model.md)*

[^1]: Scala: `core/shared/src/main/scala/sigma/data/AvlTreeData.scala:43-57` (AvlTreeData case class)

[^2]: Rust: `ergotree-ir/src/mir/avl_tree_data.rs:56-69` (AvlTreeData struct)

[^3]: Scala: `core/shared/src/main/scala/sigma/data/AvlTreeData.scala:57` (DigestSize = 33)

[^4]: Rust: `ergotree-ir/src/mir/avl_tree_data.rs:61-62` (digest field)

[^5]: Scala: `core/shared/src/main/scala/sigma/data/AvlTreeData.scala:7-36` (AvlTreeFlags)

[^6]: Rust: `ergotree-ir/src/mir/avl_tree_data.rs:10-54` (AvlTreeFlags impl)

[^7]: Scala: `core/shared/src/main/scala/sigma/SigmaDsl.scala:547-589` (AvlTree trait)

[^8]: Scala: `data/shared/src/main/scala/sigma/eval/AvlTreeVerifier.scala:8-88` (AvlTreeVerifier)

[^9]: Scala: `interpreter/shared/src/main/scala/sigmastate/eval/CAvlTreeVerifier.scala:17-45` (CAvlTreeVerifier)

[^10]: Scala: `interpreter/shared/src/main/scala/sigmastate/eval/CErgoTreeEvaluator.scala:78-93` (contains_eval)

[^11]: Scala: `interpreter/shared/src/main/scala/sigmastate/eval/CErgoTreeEvaluator.scala:132-164` (insert_eval)

[^12]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala:1498-1540` (cost info constants)

[^13]: Scala: `core/shared/src/main/scala/sigma/data/AvlTreeData.scala:71-90` (serializer)

[^14]: Rust: `ergotree-ir/src/mir/avl_tree_data.rs:71-91` (SigmaSerializable impl)

[^15]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala:1588` (getMany key ordering caution)