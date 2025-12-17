# Chapter 10: Hash Functions

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Cryptographic hash function properties: collision resistance, preimage resistance, deterministic output
- Understanding of message authentication codes (MACs) and their role in key derivation
- Prior chapters: [Chapter 9](./ch09-elliptic-curve-cryptography.md) for the cryptographic context, [Chapter 5](../part2/ch05-operations-opcodes.md) for opcode-based operations

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why BLAKE2b256 is the primary hash function in Ergo and when SHA-256 is used
- Implement hash operations with per-item costing based on block size
- Describe Fiat-Shamir challenge generation and why challenges are truncated to 192 bits
- Use HMAC-SHA512 for BIP32/BIP39 key derivation

## Hash Functions in Sigma

Hash functions are fundamental to blockchain security—they provide integrity guarantees, enable content addressing, and transform interactive proofs into non-interactive ones via the Fiat-Shamir heuristic. The Sigma protocol uses two primary hash functions, each optimized for different use cases[^1][^2]:

```
Hash Function Uses
─────────────────────────────────────────────────────
Purpose              Function        Output
─────────────────────────────────────────────────────
Script hashing       blake2b256()    32 bytes
External compat      sha256()        32 bytes
Challenge gen        Fiat-Shamir     24 bytes (truncated)
Box identification   blake2b256()    32 bytes
Key derivation       HMAC-SHA512     64 bytes
```

## BLAKE2b256

The primary hash function for Ergo—faster and more secure than SHA-256[^3][^4].

### Implementation

```zig
const Blake2b256 = struct {
    /// Output size in bytes
    pub const DIGEST_SIZE: usize = 32;
    /// Block size for cost calculation
    pub const BLOCK_SIZE: usize = 128;

    state: [8]u64,
    buf: [BLOCK_SIZE]u8,
    buf_len: usize,
    total_len: u128,

    const IV: [8]u64 = .{
        0x6a09e667f3bcc908, 0xbb67ae8584caa73b,
        0x3c6ef372fe94f82b, 0xa54ff53a5f1d36f1,
        0x510e527fade682d1, 0x9b05688c2b3e6c1f,
        0x1f83d9abfb41bd6b, 0x5be0cd19137e2179,
    };

    pub fn init() Blake2b256 {
        var self = Blake2b256{
            .state = IV,
            .buf = undefined,
            .buf_len = 0,
            .total_len = 0,
        };
        // Parameter block XOR (digest length, fanout, depth)
        self.state[0] ^= 0x01010000 ^ DIGEST_SIZE;
        return self;
    }

    pub fn update(self: *Blake2b256, data: []const u8) void {
        var offset: usize = 0;

        // Fill buffer if partially full
        if (self.buf_len > 0 and self.buf_len + data.len > BLOCK_SIZE) {
            const fill = BLOCK_SIZE - self.buf_len;
            @memcpy(self.buf[self.buf_len..][0..fill], data[0..fill]);
            self.compress(false);
            self.buf_len = 0;
            offset = fill;
        }

        // Process full blocks
        while (offset + BLOCK_SIZE <= data.len) {
            @memcpy(&self.buf, data[offset..][0..BLOCK_SIZE]);
            self.compress(false);
            offset += BLOCK_SIZE;
        }

        // Buffer remaining
        const remaining = data.len - offset;
        if (remaining > 0) {
            @memcpy(self.buf[self.buf_len..][0..remaining], data[offset..][0..remaining]);
            self.buf_len += remaining;
        }
        self.total_len += data.len;
    }

    pub fn final(self: *Blake2b256) [DIGEST_SIZE]u8 {
        // Pad with zeros
        @memset(self.buf[self.buf_len..], 0);
        self.compress(true); // Final block

        var result: [DIGEST_SIZE]u8 = undefined;
        for (self.state[0..4], 0..) |s, i| {
            @memcpy(result[i * 8 ..][0..8], &std.mem.toBytes(std.mem.nativeToLittle(u64, s)));
        }
        return result;
    }

    fn compress(self: *Blake2b256, is_final: bool) void {
        // BLAKE2b compression function
        // ... (standard BLAKE2b round function)
        _ = is_final;
    }

    /// One-shot hash
    pub fn hash(data: []const u8) [DIGEST_SIZE]u8 {
        var hasher = init();
        hasher.update(data);
        return hasher.final();
    }
};
```

### AST Node

```zig
const CalcBlake2b256 = struct {
    input: *const Expr, // Coll[Byte]

    pub const OP_CODE = OpCode.new(87);

    pub const COST = PerItemCost{
        .base = JitCost{ .value = 20 },
        .per_chunk = JitCost{ .value = 7 },
        .chunk_size = 128,
    };

    pub fn tpe(_: *const CalcBlake2b256) SType {
        return .{ .coll = &SType.byte };
    }

    pub fn eval(self: *const CalcBlake2b256, env: *const DataEnv, E: *Evaluator) ![]const u8 {
        const input_bytes = try self.input.eval(env, E);
        const coll = input_bytes.coll.bytes;

        // Add cost based on input length
        E.addSeqCost(COST, coll.len, OP_CODE);

        const result = Blake2b256.hash(coll);
        return try E.allocator.dupe(u8, &result);
    }
};
```

## SHA-256

Available for external system compatibility (Bitcoin, etc.)[^5][^6].

### Implementation

```zig
const Sha256 = struct {
    pub const DIGEST_SIZE: usize = 32;
    pub const BLOCK_SIZE: usize = 64;

    state: [8]u32,
    buf: [BLOCK_SIZE]u8,
    buf_len: usize,
    total_len: u64,

    const K: [64]u32 = .{
        0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
        0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
        // ... remaining round constants
    };

    const H0: [8]u32 = .{
        0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
        0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19,
    };

    pub fn init() Sha256 {
        return .{
            .state = H0,
            .buf = undefined,
            .buf_len = 0,
            .total_len = 0,
        };
    }

    pub fn hash(data: []const u8) [DIGEST_SIZE]u8 {
        var hasher = init();
        hasher.update(data);
        return hasher.final();
    }

    // ... update, final, compress methods
};
```

### AST Node

```zig
const CalcSha256 = struct {
    input: *const Expr,

    pub const OP_CODE = OpCode.new(88);

    /// SHA-256 is more expensive than BLAKE2b
    pub const COST = PerItemCost{
        .base = JitCost{ .value = 80 },
        .per_chunk = JitCost{ .value = 8 },
        .chunk_size = 64,
    };

    pub fn eval(self: *const CalcSha256, env: *const DataEnv, E: *Evaluator) ![]const u8 {
        const input_bytes = try self.input.eval(env, E);
        const coll = input_bytes.coll.bytes;

        E.addSeqCost(COST, coll.len, OP_CODE);

        const result = Sha256.hash(coll);
        return try E.allocator.dupe(u8, &result);
    }
};
```

## Cost Comparison

```
Hash Function Costs
─────────────────────────────────────────────────────
             Base    Per Chunk   Chunk Size
─────────────────────────────────────────────────────
BLAKE2b256   20      7           128 bytes
SHA-256      80      8           64 bytes
─────────────────────────────────────────────────────

Cost Formula:  total = base + ceil(len / chunk_size) * per_chunk
```

### Example: 200-byte Input

```
BLAKE2b256:
  chunks = ceil(200 / 128) = 2
  cost = 20 + 2 * 7 = 34

SHA-256:
  chunks = ceil(200 / 64) = 4
  cost = 80 + 4 * 8 = 112

Ratio: SHA-256 is ~3.3x more expensive
```

## Fiat-Shamir Hash

Internal hash for Sigma protocol challenge generation[^7][^8]:

```zig
const FiatShamir = struct {
    /// Soundness bits (192 = 24 bytes)
    pub const SOUNDNESS_BITS: u32 = 192;
    pub const SOUNDNESS_BYTES: usize = SOUNDNESS_BITS / 8; // 24

    /// Fiat-Shamir hash function
    /// Returns first 24 bytes of BLAKE2b256 hash
    pub fn hashFn(input: []const u8) [SOUNDNESS_BYTES]u8 {
        const full_hash = Blake2b256.hash(input);

        var result: [SOUNDNESS_BYTES]u8 = undefined;
        @memcpy(&result, full_hash[0..SOUNDNESS_BYTES]);
        return result;
    }
};
```

### Why 192 Bits?

The truncation to 192 bits is not arbitrary[^9]:

```
Security Constraints
─────────────────────────────────────────────────────
1. Challenge must be unpredictable to cheating prover
2. Threshold signatures use GF(2^192) polynomials
3. Must satisfy: 2^soundnessBits < group_order
4. Group order ≈ 2^256, so 192 < 256 works
```

```zig
comptime {
    // This constraint is critical for security
    std.debug.assert(FiatShamir.SOUNDNESS_BITS < CryptoConstants.GROUP_SIZE_BITS);
}
```

## Fiat-Shamir Tree Serialization

The challenge is computed from a serialized proof tree[^10]:

```zig
const FiatShamirTreeSerializer = struct {
    const INTERNAL_NODE_PREFIX: u8 = 0;
    const LEAF_PREFIX: u8 = 1;

    pub fn serialize(tree: *const ProofTree, writer: anytype) !void {
        switch (tree.*) {
            .leaf => |leaf| {
                try writer.writeByte(LEAF_PREFIX);

                // Serialize proposition as ErgoTree
                const prop_bytes = try leaf.proposition.toErgoTreeBytes();
                try writer.writeInt(i16, @intCast(prop_bytes.len), .big);
                try writer.writeAll(prop_bytes);

                // Serialize commitment
                const commitment = leaf.commitment orelse
                    return error.EmptyCommitment;
                try writer.writeInt(i16, @intCast(commitment.len), .big);
                try writer.writeAll(commitment);
            },
            .conjecture => |conj| {
                try writer.writeByte(INTERNAL_NODE_PREFIX);
                try writer.writeByte(@intFromEnum(conj.conj_type));

                // Threshold k for CTHRESHOLD
                if (conj.conj_type == .cthreshold) {
                    try writer.writeByte(conj.k);
                }

                try writer.writeInt(i16, @intCast(conj.children.len), .big);
                for (conj.children) |child| {
                    try serialize(child, writer);
                }
            },
        }
    }
};
```

## HMAC-SHA512

For BIP32/BIP39 key derivation[^11]:

```zig
const HmacSha512 = struct {
    pub const DIGEST_SIZE: usize = 64;
    pub const BLOCK_SIZE: usize = 128;

    inner: Sha512,
    outer: Sha512,

    pub fn init(key: []const u8) HmacSha512 {
        var padded_key: [BLOCK_SIZE]u8 = [_]u8{0} ** BLOCK_SIZE;

        if (key.len > BLOCK_SIZE) {
            const hashed = Sha512.hash(key);
            @memcpy(padded_key[0..64], &hashed);
        } else {
            @memcpy(padded_key[0..key.len], key);
        }

        // Inner padding (0x36)
        var inner_pad: [BLOCK_SIZE]u8 = undefined;
        for (padded_key, 0..) |b, i| {
            inner_pad[i] = b ^ 0x36;
        }

        // Outer padding (0x5c)
        var outer_pad: [BLOCK_SIZE]u8 = undefined;
        for (padded_key, 0..) |b, i| {
            outer_pad[i] = b ^ 0x5c;
        }

        var self = HmacSha512{
            .inner = Sha512.init(),
            .outer = Sha512.init(),
        };
        self.inner.update(&inner_pad);
        self.outer.update(&outer_pad);
        return self;
    }

    pub fn update(self: *HmacSha512, data: []const u8) void {
        self.inner.update(data);
    }

    pub fn final(self: *HmacSha512) [DIGEST_SIZE]u8 {
        const inner_hash = self.inner.final();
        self.outer.update(&inner_hash);
        return self.outer.final();
    }

    pub fn hash(key: []const u8, data: []const u8) [DIGEST_SIZE]u8 {
        var hmac = init(key);
        hmac.update(data);
        return hmac.final();
    }
};
```

### Key Derivation Constants

```zig
const KeyDerivation = struct {
    /// BIP39 HMAC key
    pub const BITCOIN_SEED = "Bitcoin seed";

    /// PBKDF2 iterations for BIP39
    pub const PBKDF2_ITERATIONS: u32 = 2048;

    /// Derived key length
    pub const PBKDF2_KEY_LENGTH: u32 = 512;
};
```

## Box ID Computation

Box IDs are BLAKE2b256 hashes of box content:

```zig
pub fn computeBoxId(box_bytes: []const u8) [32]u8 {
    return Blake2b256.hash(box_bytes);
}
```

## Summary

This chapter covered the hash functions that provide cryptographic integrity throughout the Sigma protocol:

- **BLAKE2b256** is the primary hash function—approximately 3x cheaper than SHA-256 due to its larger block size (128 bytes vs 64 bytes) and optimized design
- **SHA-256** is available for external system compatibility (Bitcoin scripts, cross-chain verification)
- **Fiat-Shamir challenge generation** uses BLAKE2b256 truncated to 192 bits, matching the threshold signature polynomial field size while satisfying the constraint 2^192 < group_order
- **Per-item costing** calculates hash cost as `base + ceil(input_length / block_size) * per_chunk`, accurately reflecting the computational work
- **HMAC-SHA512** provides key derivation for BIP32/BIP39 wallet compatibility, using the standard "Bitcoin seed" key
- **Box IDs** are computed as BLAKE2b256 hashes of serialized box content, providing content-addressable identification

---

*Next: [Chapter 11: Sigma Protocols](./ch11-sigma-protocols.md)*

[^1]: Scala: [`CryptoFunctions.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/crypto/CryptoFunctions.scala)

[^2]: Rust: [`hash.rs:5-26`](https://github.com/ergoplatform/sigma-rust/blob/develop/sigma-util/src/hash.rs#L5-L26)

[^3]: Scala: [`trees.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/trees.scala) (CalcBlake2b256)

[^4]: Rust: [`calc_blake2b256.rs:14-47`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/calc_blake2b256.rs#L14-L47)

[^5]: Scala: [`trees.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/trees.scala) (CalcSha256)

[^6]: Rust: [`calc_sha256.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/calc_sha256.rs)

[^7]: Scala: [`CryptoFunctions.scala:hashFn`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/crypto/CryptoFunctions.scala)

[^8]: Rust: [`fiat_shamir.rs:70-76`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/fiat_shamir.rs#L70-L76)

[^9]: Scala: [`CryptoConstants.scala:70-75`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/crypto/CryptoConstants.scala#L70-L75)

[^10]: Rust: [`fiat_shamir.rs:116-203`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/fiat_shamir.rs#L116-L203)

[^11]: Scala: [`HmacSHA512.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/jvm/src/main/scala/sigma/crypto/HmacSHA512.scala)