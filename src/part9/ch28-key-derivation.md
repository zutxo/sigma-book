# Chapter 28: Key Derivation

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Elliptic curve cryptography ([Chapter 9](../part4/ch09-elliptic-curve-cryptography.md))
- Hash functions ([Chapter 10](../part4/ch10-hash-functions.md))
- High-level SDK ([Chapter 27](./ch27-high-level-sdk.md))

## Learning Objectives

- Understand BIP-32 hierarchical deterministic key derivation
- Implement derivation paths and index encoding
- Distinguish hardened from non-hardened derivation
- Master EIP-3 key derivation for Ergo

## HD Wallet Architecture

Hierarchical Deterministic (HD) wallets derive unlimited keys from a single master seed[^1][^2]:

```
HD Key Derivation Tree
══════════════════════════════════════════════════════════════════

                     Master Seed (BIP-39)
                            │
                     HMAC-SHA512("Bitcoin seed", seed)
                            │
              ┌─────────────┴─────────────┐
              │                           │
         Master Key                  Chain Code
         (32 bytes)                  (32 bytes)
              │                           │
              └───────────┬───────────────┘
                          │
                   Extended Master Key
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    m/44' (Purpose)  m/44'/429'      m/44'/429'/0'
         │           (Coin Type)      (Account)
         │                │                │
         ▼                ▼                ▼
    BIP-44 Keys      Ergo Keys       Account Keys
```

## Index Types

Child indices distinguish hardened from normal derivation[^3][^4]:

```zig
const ChildIndex = union(enum) {
    hardened: HardenedIndex,
    normal: NormalIndex,

    const HardenedIndex = struct {
        value: u31, // 0 to 2^31-1

        pub fn toBits(self: HardenedIndex) u32 {
            return @as(u32, self.value) | HARDENED_BIT;
        }
    };

    const NormalIndex = struct {
        value: u31, // 0 to 2^31-1

        pub fn toBits(self: NormalIndex) u32 {
            return @as(u32, self.value);
        }

        pub fn next(self: NormalIndex) NormalIndex {
            return .{ .value = self.value + 1 };
        }
    };

    const HARDENED_BIT: u32 = 0x80000000; // 2^31

    pub fn hardened(i: u31) ChildIndex {
        return .{ .hardened = .{ .value = i } };
    }

    pub fn normal(i: u31) ChildIndex {
        return .{ .normal = .{ .value = i } };
    }

    pub fn toBits(self: ChildIndex) u32 {
        return switch (self) {
            .hardened => |h| h.toBits(),
            .normal => |n| n.toBits(),
        };
    }

    pub fn isHardened(self: ChildIndex) bool {
        return self == .hardened;
    }
};
```

## Hardened vs Normal Derivation

```
Derivation Security Properties
══════════════════════════════════════════════════════════════════

┌──────────────┬─────────────────┬─────────────────────────────────┐
│ Type         │ Index Range     │ Security Property               │
├──────────────┼─────────────────┼─────────────────────────────────┤
│ Normal       │ 0 to 2³¹-1      │ Public derivation possible      │
│              │ (0, 1, 2)       │ Child pubkey from parent pubkey │
├──────────────┼─────────────────┼─────────────────────────────────┤
│ Hardened     │ 2³¹ to 2³²-1    │ Requires private key            │
│              │ (0', 1', 2')    │ Prevents key leakage            │
└──────────────┴─────────────────┴─────────────────────────────────┘

Why Hardened Matters:
─────────────────────────────────────────────────────────────────
If attacker obtains:
  - Child private key (leaked)
  - Parent chain code (public in xpub)

With normal derivation: Attacker can compute parent private key!
With hardened derivation: Parent key remains secure
```

## Derivation Path

Paths encode the key tree location[^5][^6]:

```zig
const DerivationPath = struct {
    indices: []const ChildIndex,

    const PURPOSE: ChildIndex = ChildIndex.hardened(44);
    const ERG_COIN_TYPE: ChildIndex = ChildIndex.hardened(429);
    const CHANGE_EXTERNAL: ChildIndex = ChildIndex.normal(0);

    /// Create EIP-3 compliant path: m/44'/429'/account'/0/address
    pub fn eip3(account: u31, address: u31) DerivationPath {
        return .{
            .indices = &[_]ChildIndex{
                PURPOSE,
                ERG_COIN_TYPE,
                ChildIndex.hardened(account),
                CHANGE_EXTERNAL,
                ChildIndex.normal(address),
            },
        };
    }

    /// Master path (empty)
    pub fn master() DerivationPath {
        return .{ .indices = &[_]ChildIndex{} };
    }

    pub fn depth(self: *const DerivationPath) usize {
        return self.indices.len;
    }

    /// Extend path with new index
    pub fn extend(self: *const DerivationPath, index: ChildIndex, allocator: Allocator) !DerivationPath {
        var new_indices = try allocator.alloc(ChildIndex, self.indices.len + 1);
        @memcpy(new_indices[0..self.indices.len], self.indices);
        new_indices[self.indices.len] = index;
        return .{ .indices = new_indices };
    }

    /// Increment last index
    pub fn next(self: *const DerivationPath, allocator: Allocator) !DerivationPath {
        if (self.indices.len == 0) return error.EmptyPath;

        var new_indices = try allocator.dupe(ChildIndex, self.indices);
        const last = &new_indices[new_indices.len - 1];
        last.* = switch (last.*) {
            .hardened => |h| ChildIndex.hardened(h.value + 1),
            .normal => |n| ChildIndex.normal(n.value + 1),
        };
        return .{ .indices = new_indices };
    }
};
```

## Path Parsing and Display

```zig
const PathParser = struct {
    pub fn parse(path_str: []const u8, allocator: Allocator) !DerivationPath {
        var indices = std.ArrayList(ChildIndex).init(allocator);

        var iter = std.mem.splitScalar(u8, path_str, '/');

        // First element must be 'm' or 'M'
        const master = iter.next() orelse return error.EmptyPath;
        if (!std.mem.eql(u8, master, "m") and !std.mem.eql(u8, master, "M")) {
            return error.InvalidMasterPrefix;
        }

        while (iter.next()) |segment| {
            const is_hardened = std.mem.endsWith(u8, segment, "'");
            const num_str = if (is_hardened)
                segment[0 .. segment.len - 1]
            else
                segment;

            const value = try std.fmt.parseInt(u31, num_str, 10);
            const index = if (is_hardened)
                ChildIndex.hardened(value)
            else
                ChildIndex.normal(value);

            try indices.append(index);
        }

        return .{ .indices = try indices.toOwnedSlice() };
    }

    pub fn format(path: *const DerivationPath, writer: anytype) !void {
        try writer.writeAll("m");
        for (path.indices) |index| {
            try writer.writeAll("/");
            switch (index) {
                .hardened => |h| try writer.print("{}'", .{h.value}),
                .normal => |n| try writer.print("{}", .{n.value}),
            }
        }
    }
};
```

## EIP-3 Derivation Standard

Ergo's EIP-3 defines the derivation structure[^7][^8]:

```
EIP-3 Path Structure
══════════════════════════════════════════════════════════════════

m / 44' / 429' / account' / change / address
│    │      │        │         │        │
│    │      │        │         │        └── Address Index (normal)
│    │      │        │         └─────────── Change: 0=external, 1=internal
│    │      │        └───────────────────── Account Index (hardened)
│    │      └────────────────────────────── Coin Type: 429 (Ergo)
│    └───────────────────────────────────── Purpose: BIP-44
└────────────────────────────────────────── Master private key

Examples:
  m/44'/429'/0'/0/0   First address, first account
  m/44'/429'/0'/0/1   Second address, first account
  m/44'/429'/1'/0/0   First address, second account
```

## Extended Secret Key

Extended keys pair key material with chain code[^9][^10]:

```zig
const ExtSecretKey = struct {
    key_bytes: [32]u8,      // Private key scalar
    chain_code: [32]u8,     // Chain code for derivation
    path: DerivationPath,

    const BITCOIN_SEED = "Bitcoin seed";

    /// Derive master key from seed
    pub fn deriveMaster(seed: []const u8) !ExtSecretKey {
        var hmac = HmacSha512.init(BITCOIN_SEED);
        hmac.update(seed);
        var output: [64]u8 = undefined;
        hmac.final(&output);

        return ExtSecretKey{
            .key_bytes = output[0..32].*,
            .chain_code = output[32..64].*,
            .path = DerivationPath.master(),
        };
    }

    /// Get public image (ProveDlog)
    pub fn publicImage(self: *const ExtSecretKey) ProveDlog {
        const scalar = Scalar.fromBytes(self.key_bytes);
        const point = CryptoConstants.generator.mul(scalar);
        return ProveDlog{ .h = point };
    }

    /// Get corresponding extended public key
    pub fn publicKey(self: *const ExtSecretKey) !ExtPubKey {
        return ExtPubKey{
            .key_bytes = self.publicImage().compress(),
            .chain_code = self.chain_code,
            .path = self.path,
        };
    }

    /// Zero out key material
    pub fn zeroSecret(self: *ExtSecretKey) void {
        @memset(&self.key_bytes, 0);
    }
};
```

## Child Key Derivation

BIP-32 child derivation algorithm[^11][^12]:

```zig
pub fn deriveChild(parent: *const ExtSecretKey, index: ChildIndex, allocator: Allocator) !ExtSecretKey {
    var hmac = HmacSha512.init(&parent.chain_code);

    // HMAC input depends on derivation type
    switch (index) {
        .hardened => {
            // Hardened: 0x00 || parent_key (33 bytes)
            hmac.update(&[_]u8{0x00});
            hmac.update(&parent.key_bytes);
        },
        .normal => {
            // Normal: parent_public_key (33 bytes compressed)
            const pub_key = parent.publicImage().compress();
            hmac.update(&pub_key);
        },
    }

    // Append index as big-endian u32
    var index_bytes: [4]u8 = undefined;
    std.mem.writeInt(u32, &index_bytes, index.toBits(), .big);
    hmac.update(&index_bytes);

    var output: [64]u8 = undefined;
    hmac.final(&output);

    // Parse left 32 bytes as scalar
    const child_key_proto = Scalar.fromBytes(output[0..32].*);

    // Check validity (must be < group order)
    if (child_key_proto.isOverflow()) {
        return deriveChild(parent, index.next(), allocator);
    }

    // child_key = (child_key_proto + parent_key) mod n
    const parent_scalar = Scalar.fromBytes(parent.key_bytes);
    const child_scalar = child_key_proto.add(parent_scalar);

    // Check for zero (invalid)
    if (child_scalar.isZero()) {
        return deriveChild(parent, index.next(), allocator);
    }

    return ExtSecretKey{
        .key_bytes = child_scalar.toBytes(),
        .chain_code = output[32..64].*,
        .path = try parent.path.extend(index, allocator),
    };
}

/// Derive key at full path
pub fn derive(master: *const ExtSecretKey, path: DerivationPath, allocator: Allocator) !ExtSecretKey {
    var current = master.*;
    for (path.indices) |index| {
        current = try deriveChild(&current, index, allocator);
    }
    return current;
}
```

## Extended Public Key

Public key derivation (non-hardened only)[^13][^14]:

```zig
const ExtPubKey = struct {
    key_bytes: [33]u8,      // Compressed public key
    chain_code: [32]u8,
    path: DerivationPath,

    pub fn deriveChild(parent: *const ExtPubKey, index: ChildIndex, allocator: Allocator) !ExtPubKey {
        // Cannot derive hardened children from public key
        if (index.isHardened()) {
            return error.HardenedDerivationRequiresPrivateKey;
        }

        var hmac = HmacSha512.init(&parent.chain_code);
        hmac.update(&parent.key_bytes);

        var index_bytes: [4]u8 = undefined;
        std.mem.writeInt(u32, &index_bytes, index.toBits(), .big);
        hmac.update(&index_bytes);

        var output: [64]u8 = undefined;
        hmac.final(&output);

        const child_key_proto = Scalar.fromBytes(output[0..32].*);

        if (child_key_proto.isOverflow()) {
            return deriveChild(parent, index.next(), allocator);
        }

        // child_public = point(child_key_proto) + parent_public
        const proto_point = CryptoConstants.generator.mul(child_key_proto);
        const parent_point = Point.decompress(parent.key_bytes);
        const child_point = proto_point.add(parent_point);

        if (child_point.isInfinity()) {
            return deriveChild(parent, index.next(), allocator);
        }

        return ExtPubKey{
            .key_bytes = child_point.compress(),
            .chain_code = output[32..64].*,
            .path = try parent.path.extend(index, allocator),
        };
    }
};
```

## Mnemonic to Seed

BIP-39 seed derivation[^15][^16]:

```zig
const Mnemonic = struct {
    const PBKDF2_ITERATIONS: u32 = 2048;
    const SEED_LENGTH: usize = 64;

    /// Convert mnemonic phrase to seed using PBKDF2-HMAC-SHA512
    pub fn toSeed(phrase: []const u8, passphrase: []const u8) [SEED_LENGTH]u8 {
        var seed: [SEED_LENGTH]u8 = undefined;

        // Normalize using NFKD
        const normalized_phrase = normalizeNfkd(phrase);
        const normalized_pass = normalizeNfkd(passphrase);

        // Salt = "mnemonic" + passphrase
        var salt_buf: [256]u8 = undefined;
        const salt = std.fmt.bufPrint(&salt_buf, "mnemonic{s}", .{normalized_pass}) catch unreachable;

        // PBKDF2-HMAC-SHA512
        pbkdf2(
            HmacSha512,
            normalized_phrase,
            salt,
            PBKDF2_ITERATIONS,
            &seed,
        );

        return seed;
    }
};

/// Full derivation from mnemonic to key
pub fn mnemonicToKey(
    phrase: []const u8,
    passphrase: []const u8,
    path: DerivationPath,
    allocator: Allocator,
) !ExtSecretKey {
    const seed = Mnemonic.toSeed(phrase, passphrase);
    const master = try ExtSecretKey.deriveMaster(&seed);
    return derive(&master, path, allocator);
}
```

## Path Serialization

Binary format for storage/transfer[^17][^18]:

```zig
const DerivationPathSerializer = struct {
    pub fn serialize(path: *const DerivationPath, writer: anytype) !void {
        // Public branch flag (0x00 for private, 0x01 for public)
        try writer.writeByte(0x00);

        // Depth
        try writer.writeInt(u32, @intCast(path.indices.len), .little);

        // Each index as 4-byte big-endian
        for (path.indices) |index| {
            var bytes: [4]u8 = undefined;
            std.mem.writeInt(u32, &bytes, index.toBits(), .big);
            try writer.writeAll(&bytes);
        }
    }

    pub fn parse(reader: anytype, allocator: Allocator) !DerivationPath {
        const public_branch = try reader.readByte();
        _ = public_branch; // TODO: handle public branch

        const depth = try reader.readInt(u32, .little);

        var indices = try allocator.alloc(ChildIndex, depth);
        for (0..depth) |i| {
            var bytes: [4]u8 = undefined;
            try reader.readNoEof(&bytes);
            const bits = std.mem.readInt(u32, &bytes, .big);

            indices[i] = if (bits & 0x80000000 != 0)
                ChildIndex.hardened(@truncate(bits & 0x7FFFFFFF))
            else
                ChildIndex.normal(@truncate(bits));
        }

        return .{ .indices = indices };
    }
};
```

## Watch-Only Wallet

Public key derivation enables watch-only wallets:

```
Watch-Only Wallet Setup
══════════════════════════════════════════════════════════════════

Full Wallet (has secrets)           Watch-Only Wallet (no secrets)
─────────────────────────           ──────────────────────────────

Master Secret Key
       │
       ├── m/44'/429'/0'            Extended Public Key
       │   (hardened account)  ───▶  at m/44'/429'/0'/0
       │          │                        │
       │          └── m/44'/429'/0'/0      ├── Address 0 public
       │              (change branch) ───▶ ├── Address 1 public
       │                   │               ├── Address 2 public
       │                   ├── 0           └── ... (can derive more)
       │                   ├── 1
       │                   └── 2           Cannot derive:
       │                                    × Account 1 keys
                                            × Hardened children
                                            × Private keys

Export at: m/44'/429'/0'/0 (parent of address keys)
Can derive: All non-hardened children (addresses)
Cannot derive: Hardened children, private keys
```

## Usage Example

```zig
const allocator = std.heap.page_allocator;

// 1. From mnemonic to master key
const mnemonic = "abandon abandon abandon abandon abandon abandon " ++
                 "abandon abandon abandon abandon abandon about";
const seed = Mnemonic.toSeed(mnemonic, "");
var master = try ExtSecretKey.deriveMaster(&seed);
defer master.zeroSecret();

// 2. Derive first EIP-3 address key
const path = DerivationPath.eip3(0, 0); // m/44'/429'/0'/0/0
var first_key = try derive(&master, path, allocator);
defer first_key.zeroSecret();

// 3. Get public image for address
const pub_key = first_key.publicImage();

// 4. Derive next address
const next_path = try path.next(allocator);
var second_key = try derive(&master, next_path, allocator);
defer second_key.zeroSecret();

// 5. Create watch-only wallet
const watch_only_path = try PathParser.parse("m/44'/429'/0'/0", allocator);
var account_key = try derive(&master, watch_only_path, allocator);
const watch_only = try account_key.publicKey();

// 6. Derive address public keys without secrets
const addr0_pub = try watch_only.deriveChild(ChildIndex.normal(0), allocator);
const addr1_pub = try watch_only.deriveChild(ChildIndex.normal(1), allocator);

// 7. Cannot derive hardened from public key
_ = watch_only.deriveChild(ChildIndex.hardened(0), allocator) catch |err| {
    std.debug.assert(err == error.HardenedDerivationRequiresPrivateKey);
};
```

## Security Considerations

```
Key Derivation Security
══════════════════════════════════════════════════════════════════

Attack: Child + Chain Code → Parent
────────────────────────────────────
Given:
  - Child private key k_i
  - Parent chain code c

For NORMAL derivation:
  HMAC-SHA512(c, K_parent || i) = IL || IR
  k_i = IL + k_parent  mod n

  Attacker can compute:
  k_parent = k_i - IL  mod n  ← COMPROMISED!

For HARDENED derivation:
  HMAC-SHA512(c, 0x00 || k_parent || i) = IL || IR

  Cannot compute IL without knowing k_parent
  → Parent key remains SECURE

Recommendation:
  └── Always use hardened derivation for account/purpose levels
  └── Normal derivation only for address indices
```

## Summary

- **BIP-32** defines hierarchical deterministic key derivation
- **Derivation paths** use notation `m/44'/429'/0'/0/0`
- **Hardened derivation** (`'`) requires private key; prevents key leakage
- **Normal derivation** allows public key derivation from parent public key
- **EIP-3** standardizes Ergo's path: `m/44'/429'/account'/change/address`
- **Extended keys** = key material (32 bytes) + chain code (32 bytes)
- **Watch-only wallets** use extended public keys for address generation

---

*Next: [Chapter 29: Soft Fork Mechanism](../part10/ch29-soft-fork-mechanism.md)*

[^1]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedSecretKey.scala`

[^2]: Rust: `ergo-lib/src/wallet/ext_secret_key.rs:29-37`

[^3]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/Index.scala:5-16`

[^4]: Rust: `ergo-lib/src/wallet/derivation_path.rs:15-131`

[^5]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/DerivationPath.scala:10-29`

[^6]: Rust: `ergo-lib/src/wallet/derivation_path.rs:133-204`

[^7]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/Constants.scala:31-36`

[^8]: Rust: `ergo-lib/src/wallet/derivation_path.rs:88-91` (PURPOSE, ERG, CHANGE constants)

[^9]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedSecretKey.scala:13-49`

[^10]: Rust: `ergo-lib/src/wallet/ext_secret_key.rs:60-112`

[^11]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedSecretKey.scala:53-78`

[^12]: Rust: `ergo-lib/src/wallet/ext_secret_key.rs:114-163`

[^13]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedPublicKey.scala:46-59`

[^14]: Rust: `ergo-lib/src/wallet/ext_pub_key.rs`

[^15]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/JavaHelpers.scala:282-301`

[^16]: Rust: `ergo-lib/src/wallet/mnemonic.rs:20-37`

[^17]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/DerivationPath.scala:133-147`

[^18]: Rust: `ergo-lib/src/wallet/derivation_path.rs:235-241` (ledger_bytes)