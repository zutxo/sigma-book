# Chapter 32: v6 Protocol Features

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 2](../part1/ch02-type-system.md) for the ErgoTree type system and numeric types
- [Chapter 6](../part2/ch06-methods-on-types.md) for method definitions on types
- [Chapter 29](./ch29-soft-fork-mechanism.md) for soft-fork versioning and activation

## Learning Objectives

By the end of this chapter, you will be able to:

- Implement SUnsignedBigInt (256-bit unsigned integers) with modular arithmetic operations
- Apply bitwise operations (AND, OR, XOR, NOT, shifts) to all numeric types
- Use new collection manipulation methods (patch, updated, updateMany, reverse, get)
- Understand the Autolykos2 proof-of-work algorithm and Header.checkPow
- Serialize values using Global.serialize and decode difficulty with NBits encoding
- Write version-aware scripts that use v6 features safely

## Version Activation

ErgoTree version 3 corresponds to protocol v6.0. Features in this chapter are only available when the v6 soft-fork is activated.

```
Version Mapping
═══════════════════════════════════════════════════════════════════

Block Version    ErgoTree Version    Protocol    Features
─────────────────────────────────────────────────────────────────────
1-2              0-1                 v3.x-v4.x   AOT costing
3                2                   v5.x        JIT costing
4                3                   v6.x        This chapter
```

### Version Context

```zig
const VersionContext = struct {
    activated_version: u8,
    ergo_tree_version: u8,

    pub const JIT_ACTIVATION_VERSION: u8 = 2;   // v5.0
    pub const V6_SOFT_FORK_VERSION: u8 = 3;     // v6.0

    /// True if v6.0 protocol is activated
    pub fn isV6Activated(self: *const VersionContext) bool {
        return self.activated_version >= V6_SOFT_FORK_VERSION;
    }

    /// True if current ErgoTree is v3 or later
    pub fn isV3OrLaterErgoTree(self: *const VersionContext) bool {
        return self.ergo_tree_version >= V6_SOFT_FORK_VERSION;
    }

    /// Check if a v6 method can be used
    pub fn canUseV6Method(self: *const VersionContext) bool {
        return self.isV6Activated() and self.isV3OrLaterErgoTree();
    }
};
```

## SUnsignedBigInt Type

The `SUnsignedBigInt` type (type code 9) is a 256-bit unsigned integer designed for cryptographic modular arithmetic[^1][^2]. Unlike `SBigInt` which is signed, `SUnsignedBigInt` guarantees non-negative values—essential for operations like modular exponentiation where sign handling would introduce complexity and potential errors.

### Type Definition

```zig
/// 256-bit unsigned integer for modular arithmetic
/// Type code: 0x09
const UnsignedBigInt256 = struct {
    /// Internal representation: 4 x 64-bit words (little-endian)
    words: [4]u64,

    pub const TYPE_CODE: u8 = 0x09;
    pub const BIT_WIDTH: usize = 256;
    pub const BYTE_WIDTH: usize = 32;

    /// Maximum value: 2^256 - 1
    pub const MAX = UnsignedBigInt256{ .words = .{
        0xFFFFFFFFFFFFFFFF, 0xFFFFFFFFFFFFFFFF,
        0xFFFFFFFFFFFFFFFF, 0xFFFFFFFFFFFFFFFF,
    }};

    /// Zero value
    pub const ZERO = UnsignedBigInt256{ .words = .{ 0, 0, 0, 0 }};

    /// One value
    pub const ONE = UnsignedBigInt256{ .words = .{ 1, 0, 0, 0 }};

    /// Create from bytes (big-endian)
    pub fn fromBytes(bytes: [32]u8) UnsignedBigInt256 {
        var result = UnsignedBigInt256{ .words = undefined };
        // Convert big-endian bytes to little-endian words
        inline for (0..4) |i| {
            const offset = (3 - i) * 8;
            result.words[i] = std.mem.readInt(u64, bytes[offset..][0..8], .big);
        }
        return result;
    }

    /// Convert to bytes (big-endian)
    pub fn toBytes(self: UnsignedBigInt256) [32]u8 {
        var result: [32]u8 = undefined;
        inline for (0..4) |i| {
            const offset = (3 - i) * 8;
            std.mem.writeInt(u64, result[offset..][0..8], self.words[i], .big);
        }
        return result;
    }

    /// Convert from signed BigInt (errors if negative)
    pub fn fromBigInt(bi: BigInt256) !UnsignedBigInt256 {
        if (bi.isNegative()) {
            return error.NegativeValue;
        }
        // Safe to reinterpret since non-negative
        return @bitCast(bi.abs());
    }

    /// Convert to signed BigInt (errors if > BigInt.MAX)
    pub fn toBigInt(self: UnsignedBigInt256) !BigInt256 {
        // Check if value exceeds signed max (2^255 - 1)
        if (self.words[3] & 0x8000000000000000 != 0) {
            return error.Overflow;
        }
        return @bitCast(self);
    }

    /// Comparison
    pub fn lessThan(self: UnsignedBigInt256, other: UnsignedBigInt256) bool {
        // Compare from most significant word
        var i: usize = 4;
        while (i > 0) {
            i -= 1;
            if (self.words[i] < other.words[i]) return true;
            if (self.words[i] > other.words[i]) return false;
        }
        return false; // Equal
    }

    pub fn eql(self: UnsignedBigInt256, other: UnsignedBigInt256) bool {
        return std.mem.eql(u64, &self.words, &other.words);
    }
};
```

### Why Unsigned Matters for Cryptography

Signed integers introduce complexity in modular arithmetic:

1. **Sign bit ambiguity**: In two's complement, the high bit indicates sign. For cryptographic operations on field elements, all 256 bits should represent magnitude.

2. **Modular reduction**: Computing `a mod m` for negative `a` requires adjustment: `(-5) mod 7 = 2`, not `-5`. Unsigned values eliminate this edge case.

3. **Constant-time operations**: Sign handling can introduce timing variations. Unsigned operations are more naturally constant-time.

4. **Field element representation**: Finite field elements are inherently non-negative integers in `[0, p-1]`.

### Serialization

```zig
const UnsignedBigInt256Serializer = struct {
    /// Serialize to variable-length big-endian bytes
    pub fn serialize(value: UnsignedBigInt256, writer: anytype) !void {
        const bytes = value.toBytes();

        // Find first non-zero byte (skip leading zeros)
        var start: usize = 0;
        while (start < 32 and bytes[start] == 0) : (start += 1) {}

        // Write length + bytes
        const len = 32 - start;
        try writer.writeInt(u8, @intCast(len), .big);
        try writer.writeAll(bytes[start..]);
    }

    /// Deserialize from variable-length big-endian bytes
    pub fn deserialize(reader: anytype) !UnsignedBigInt256 {
        const len = try reader.readInt(u8, .big);
        if (len > 32) return error.InvalidLength;

        var bytes: [32]u8 = .{0} ** 32;
        const start = 32 - len;
        try reader.readNoEof(bytes[start..]);

        return UnsignedBigInt256.fromBytes(bytes);
    }
};
```

## Modular Arithmetic Operations

v6 adds six modular arithmetic methods to `SUnsignedBigInt`[^3][^4]:

```
Modular Arithmetic Methods
═══════════════════════════════════════════════════════════════════

Method          Signature                     Cost    Description
─────────────────────────────────────────────────────────────────────
mod             (UBI, UBI) → UBI              20      a mod m
modInverse      (UBI, UBI) → UBI              150     a⁻¹ mod m
plusMod         (UBI, UBI, UBI) → UBI         30      (a + b) mod m
subtractMod     (UBI, UBI, UBI) → UBI         30      (a - b) mod m
multiplyMod     (UBI, UBI, UBI) → UBI         40      (a × b) mod m
toSigned        UBI → BigInt                  10      Convert to signed
```

### Basic Modulo Operation

```zig
/// a mod m - remainder after division
/// Cost: FixedCost(20)
pub fn mod(a: UnsignedBigInt256, m: UnsignedBigInt256) !UnsignedBigInt256 {
    if (m.eql(UnsignedBigInt256.ZERO)) {
        return error.DivisionByZero;
    }

    // Use schoolbook division for 256-bit values
    // Result is always < m
    return divmod(a, m).remainder;
}
```

### Modular Inverse (Extended Euclidean Algorithm)

The modular inverse `a⁻¹ mod m` is the value `x` such that `(a × x) mod m = 1`. It exists only when `gcd(a, m) = 1`.

```zig
/// Extended Euclidean Algorithm
/// Returns x such that (a * x) ≡ 1 (mod m)
/// Cost: FixedCost(150) - most expensive modular operation
pub fn modInverse(a: UnsignedBigInt256, m: UnsignedBigInt256) !UnsignedBigInt256 {
    if (m.eql(UnsignedBigInt256.ZERO)) {
        return error.DivisionByZero;
    }
    if (a.eql(UnsignedBigInt256.ZERO)) {
        return error.NoInverse; // gcd(0, m) = m ≠ 1
    }

    // Extended Euclidean Algorithm
    // Maintains: old_s * a + old_t * m = old_r (Bézout's identity)
    var old_r = a;
    var r = m;
    var old_s = UnsignedBigInt256.ONE;
    var s = UnsignedBigInt256.ZERO;
    var old_s_negative = false;
    var s_negative = false;

    while (!r.eql(UnsignedBigInt256.ZERO)) {
        const quotient = divmod(old_r, r).quotient;

        // (old_r, r) = (r, old_r - quotient * r)
        const temp_r = r;
        const qr = multiply(quotient, r);
        if (old_r.lessThan(qr)) {
            r = subtract(qr, old_r);
        } else {
            r = subtract(old_r, qr);
        }
        old_r = temp_r;

        // (old_s, s) = (s, old_s - quotient * s)
        // Handle signed arithmetic carefully
        const temp_s = s;
        const temp_s_neg = s_negative;
        const qs = multiply(quotient, s);

        if (old_s_negative == s_negative) {
            // Same sign: subtraction
            if (old_s.lessThan(qs)) {
                s = subtract(qs, old_s);
                s_negative = !old_s_negative;
            } else {
                s = subtract(old_s, qs);
                s_negative = old_s_negative;
            }
        } else {
            // Different signs: addition
            s = add(old_s, qs);
            s_negative = old_s_negative;
        }

        old_s = temp_s;
        old_s_negative = temp_s_neg;
    }

    // Check that gcd(a, m) = 1
    if (!old_r.eql(UnsignedBigInt256.ONE)) {
        return error.NoInverse; // a and m are not coprime
    }

    // Adjust result to be positive
    if (old_s_negative) {
        return subtract(m, old_s);
    }
    return old_s;
}
```

### Modular Addition

```zig
/// (a + b) mod m - modular addition
/// Handles overflow by using 320-bit intermediate
/// Cost: FixedCost(30)
pub fn plusMod(
    a: UnsignedBigInt256,
    b: UnsignedBigInt256,
    m: UnsignedBigInt256,
) !UnsignedBigInt256 {
    if (m.eql(UnsignedBigInt256.ZERO)) {
        return error.DivisionByZero;
    }

    // a + b can overflow 256 bits, so use 320-bit intermediate
    var sum: [5]u64 = .{ 0, 0, 0, 0, 0 };
    var carry: u64 = 0;

    for (0..4) |i| {
        const s = @as(u128, a.words[i]) + @as(u128, b.words[i]) + carry;
        sum[i] = @truncate(s);
        carry = @truncate(s >> 64);
    }
    sum[4] = carry;

    // Reduce mod m
    return reduce320(sum, m);
}

/// (a - b) mod m - modular subtraction
/// If a < b, result is m - (b - a)
/// Cost: FixedCost(30)
pub fn subtractMod(
    a: UnsignedBigInt256,
    b: UnsignedBigInt256,
    m: UnsignedBigInt256,
) !UnsignedBigInt256 {
    if (m.eql(UnsignedBigInt256.ZERO)) {
        return error.DivisionByZero;
    }

    if (a.lessThan(b)) {
        // a - b is negative: compute m - (b - a)
        const diff = subtract(b, a);
        const diff_mod = try mod(diff, m);
        if (diff_mod.eql(UnsignedBigInt256.ZERO)) {
            return UnsignedBigInt256.ZERO;
        }
        return subtract(m, diff_mod);
    } else {
        const diff = subtract(a, b);
        return mod(diff, m);
    }
}
```

### Modular Multiplication

```zig
/// (a * b) mod m - modular multiplication
/// Uses 512-bit intermediate to handle overflow
/// Cost: FixedCost(40)
pub fn multiplyMod(
    a: UnsignedBigInt256,
    b: UnsignedBigInt256,
    m: UnsignedBigInt256,
) !UnsignedBigInt256 {
    if (m.eql(UnsignedBigInt256.ZERO)) {
        return error.DivisionByZero;
    }

    // Multiply to 512-bit result
    var product: [8]u64 = .{0} ** 8;

    for (0..4) |i| {
        var carry: u64 = 0;
        for (0..4) |j| {
            const p = @as(u128, a.words[i]) * @as(u128, b.words[j]) +
                      @as(u128, product[i + j]) + @as(u128, carry);
            product[i + j] = @truncate(p);
            carry = @truncate(p >> 64);
        }
        product[i + 4] = carry;
    }

    // Reduce 512-bit product mod m
    return reduce512(product, m);
}
```

## Bitwise Operations

v6 adds eight bitwise methods to all numeric types (Byte, Short, Int, Long, BigInt, UnsignedBigInt)[^5][^6]:

```
Bitwise Operations
═══════════════════════════════════════════════════════════════════

Method          Signature           Cost    Description
─────────────────────────────────────────────────────────────────────
bitwiseInverse  T → T               5       ~x (NOT)
bitwiseOr       (T, T) → T          5       x | y
bitwiseAnd      (T, T) → T          5       x & y
bitwiseXor      (T, T) → T          5       x ^ y
shiftLeft       (T, Int) → T        5       x << n
shiftRight      (T, Int) → T        5       x >> n
toBytes         T → Coll[Byte]      5       Byte representation
toBits          T → Coll[Boolean]   5       Bit representation
```

### Implementation

```zig
/// Bitwise operations for all numeric types
pub fn BitwiseOps(comptime T: type) type {
    return struct {
        /// Bitwise NOT (~x)
        /// For signed types: ~x = -x - 1 (two's complement identity)
        /// For unsigned: ~x = MAX - x
        /// Cost: FixedCost(5)
        pub fn bitwiseInverse(x: T) T {
            return ~x;
        }

        /// Bitwise OR (x | y)
        /// Cost: FixedCost(5)
        pub fn bitwiseOr(x: T, y: T) T {
            return x | y;
        }

        /// Bitwise AND (x & y)
        /// Cost: FixedCost(5)
        pub fn bitwiseAnd(x: T, y: T) T {
            return x & y;
        }

        /// Bitwise XOR (x ^ y)
        /// Cost: FixedCost(5)
        pub fn bitwiseXor(x: T, y: T) T {
            return x ^ y;
        }

        /// Left shift (x << n)
        /// Returns 0 if n >= bitwidth
        /// Cost: FixedCost(5)
        pub fn shiftLeft(x: T, n: i32) !T {
            if (n < 0) return error.NegativeShift;
            const bits = @bitSizeOf(T);
            if (n >= bits) return 0;
            return x << @intCast(n);
        }

        /// Right shift (x >> n)
        /// Arithmetic shift for signed (preserves sign)
        /// Logical shift for unsigned (fills with 0)
        /// Cost: FixedCost(5)
        pub fn shiftRight(x: T, n: i32) !T {
            if (n < 0) return error.NegativeShift;
            const bits = @bitSizeOf(T);
            if (n >= bits) {
                // For signed: return -1 if negative, 0 otherwise
                // For unsigned: return 0
                if (@typeInfo(T).Int.signedness == .signed) {
                    return if (x < 0) -1 else 0;
                }
                return 0;
            }
            return x >> @intCast(n);
        }
    };
}

/// Byte conversion for numeric types
pub fn toBytes(comptime T: type, x: T) []const u8 {
    const size = @sizeOf(T);
    var result: [size]u8 = undefined;
    std.mem.writeInt(T, &result, x, .big);
    return &result;
}

/// Bit conversion for numeric types
pub fn toBits(comptime T: type, x: T) []const bool {
    const bits = @bitSizeOf(T);
    var result: [bits]bool = undefined;
    for (0..bits) |i| {
        result[bits - 1 - i] = ((x >> @intCast(i)) & 1) == 1;
    }
    return &result;
}
```

### BigInt Bitwise Operations

For `BigInt256` and `UnsignedBigInt256`, bitwise operations work on the full 256-bit representation:

```zig
/// 256-bit bitwise operations
const BigIntBitwise = struct {
    /// Bitwise NOT for 256-bit unsigned
    /// ~x = MAX - x for unsigned interpretation
    pub fn bitwiseInverse(x: UnsignedBigInt256) UnsignedBigInt256 {
        return .{ .words = .{
            ~x.words[0],
            ~x.words[1],
            ~x.words[2],
            ~x.words[3],
        }};
    }

    /// Bitwise OR for 256-bit
    pub fn bitwiseOr(x: UnsignedBigInt256, y: UnsignedBigInt256) UnsignedBigInt256 {
        return .{ .words = .{
            x.words[0] | y.words[0],
            x.words[1] | y.words[1],
            x.words[2] | y.words[2],
            x.words[3] | y.words[3],
        }};
    }

    /// Left shift for 256-bit (handles cross-word shifts)
    pub fn shiftLeft(x: UnsignedBigInt256, n: i32) !UnsignedBigInt256 {
        if (n < 0) return error.NegativeShift;
        if (n >= 256) return UnsignedBigInt256.ZERO;

        const shift: u8 = @intCast(n);
        const word_shift = shift / 64;
        const bit_shift: u6 = @intCast(shift % 64);

        var result = UnsignedBigInt256.ZERO;

        if (bit_shift == 0) {
            // Word-aligned shift
            for (word_shift..4) |i| {
                result.words[i] = x.words[i - word_shift];
            }
        } else {
            // Cross-word shift
            for (word_shift..4) |i| {
                result.words[i] = x.words[i - word_shift] << bit_shift;
                if (i > word_shift) {
                    result.words[i] |= x.words[i - word_shift - 1] >> (64 - bit_shift);
                }
            }
        }

        return result;
    }
};
```

## Collection Methods (v6)

v6 adds seven new methods to `Coll[T]` for efficient collection manipulation[^7][^8]:

```
Collection Methods (v6)
═══════════════════════════════════════════════════════════════════

Method          Signature                          Cost
─────────────────────────────────────────────────────────────────────
patch           (Coll[T], Int, Coll[T], Int) → Coll[T]  PerItem(30,2,10)
updated         (Coll[T], Int, T) → Coll[T]             PerItem(20,1,10)
updateMany      (Coll[T], Coll[Int], Coll[T]) → Coll[T] PerItem(20,2,10)
reverse         Coll[T] → Coll[T]                       PerItem (append)
startsWith      (Coll[T], Coll[T]) → Boolean            PerItem (zip)
endsWith        (Coll[T], Coll[T]) → Boolean            PerItem (zip)
get             (Coll[T], Int) → Option[T]              FixedCost(14)
```

### patch - Replace Slice

```zig
/// Replace elements from index `from`, removing `replaced` elements,
/// inserting `patch` collection in their place.
///
/// xs.patch(from, patch, replaced):
///   result = xs[0..from] ++ patch ++ xs[from+replaced..]
///
/// Cost: PerItemCost(30, 2, 10) based on xs.len + patch.len
pub fn patch(
    comptime T: type,
    xs: []const T,
    from: i32,
    patchColl: []const T,
    replaced: i32,
    allocator: Allocator,
) ![]T {
    if (from < 0) return error.IndexOutOfBounds;
    const from_idx: usize = @intCast(from);
    if (from_idx > xs.len) return error.IndexOutOfBounds;

    const replaced_count: usize = if (replaced < 0)
        0
    else
        @min(@as(usize, @intCast(replaced)), xs.len - from_idx);

    // Result length: original - replaced + patch
    const result_len = xs.len - replaced_count + patchColl.len;
    var result = try allocator.alloc(T, result_len);

    // Copy prefix [0..from]
    @memcpy(result[0..from_idx], xs[0..from_idx]);

    // Copy patch
    @memcpy(result[from_idx..][0..patchColl.len], patchColl);

    // Copy suffix [from+replaced..]
    const suffix_start = from_idx + replaced_count;
    const suffix_dest = from_idx + patchColl.len;
    @memcpy(result[suffix_dest..], xs[suffix_start..]);

    return result;
}
```

### updated - Single Element Update

```zig
/// Return a new collection with element at index replaced.
/// Immutable operation - original collection unchanged.
///
/// Cost: PerItemCost(20, 1, 10)
pub fn updated(
    comptime T: type,
    xs: []const T,
    idx: i32,
    value: T,
    allocator: Allocator,
) ![]T {
    if (idx < 0) return error.IndexOutOfBounds;
    const index: usize = @intCast(idx);
    if (index >= xs.len) return error.IndexOutOfBounds;

    var result = try allocator.dupe(T, xs);
    result[index] = value;
    return result;
}
```

### updateMany - Batch Update

```zig
/// Update multiple elements at specified indices.
/// indexes and updates must have the same length.
///
/// Cost: PerItemCost(20, 2, 10)
pub fn updateMany(
    comptime T: type,
    xs: []const T,
    indexes: []const i32,
    updates: []const T,
    allocator: Allocator,
) ![]T {
    if (indexes.len != updates.len) {
        return error.LengthMismatch;
    }

    // Validate all indexes first
    for (indexes) |idx| {
        if (idx < 0) return error.IndexOutOfBounds;
        if (@as(usize, @intCast(idx)) >= xs.len) return error.IndexOutOfBounds;
    }

    var result = try allocator.dupe(T, xs);

    for (indexes, updates) |idx, val| {
        result[@intCast(idx)] = val;
    }

    return result;
}
```

### reverse, startsWith, endsWith, get

```zig
/// Reverse collection order
/// Cost: Same as append (PerItem)
pub fn reverse(comptime T: type, xs: []const T, allocator: Allocator) ![]T {
    var result = try allocator.alloc(T, xs.len);
    for (xs, 0..) |x, i| {
        result[xs.len - 1 - i] = x;
    }
    return result;
}

/// Check if collection starts with prefix
/// Cost: Same as zip (PerItem based on prefix length)
pub fn startsWith(comptime T: type, xs: []const T, prefix: []const T) bool {
    if (prefix.len > xs.len) return false;
    return std.mem.eql(T, xs[0..prefix.len], prefix);
}

/// Check if collection ends with suffix
/// Cost: Same as zip (PerItem based on suffix length)
pub fn endsWith(comptime T: type, xs: []const T, suffix: []const T) bool {
    if (suffix.len > xs.len) return false;
    return std.mem.eql(T, xs[xs.len - suffix.len ..], suffix);
}

/// Safe element access returning Option
/// Returns null if index out of bounds (instead of error)
/// Cost: FixedCost(14)
pub fn get(comptime T: type, xs: []const T, idx: i32) ?T {
    if (idx < 0) return null;
    const index: usize = @intCast(idx);
    if (index >= xs.len) return null;
    return xs[index];
}
```

## Autolykos2 Proof-of-Work

v6 exposes proof-of-work verification in ErgoScript through `Header.checkPow()` and `Global.powHit()`[^9][^10]. Autolykos2 is Ergo's memory-hard, ASIC-resistant PoW algorithm designed for fair GPU mining.

### Algorithm Overview

```
Autolykos2 Structure
═══════════════════════════════════════════════════════════════════

Parameters:
  N = 2²⁶ ≈ 67 million     Table size (memory requirement)
  k = 32                    Elements to sum per solution
  n = 26                    log₂(N)

Memory: N × 32 bytes ≈ 2 GB table

Algorithm:
  1. Seed table from height (changes every ~1024 blocks)
  2. For each nonce attempt:
     a. Compute 32 pseudo-random indices from (msg, nonce)
     b. Sum the 32 table elements at those indices
     c. Hash (msg || nonce || sum) to get PoW hit
     d. If hit < target, solution found
```

### Implementation

```zig
/// Autolykos2 proof-of-work algorithm constants and functions
const Autolykos2 = struct {
    /// Table size: 2^26 elements
    pub const N: u32 = 1 << 26;

    /// Elements summed per solution attempt
    pub const K: u32 = 32;

    /// Bits in N (log2(N))
    pub const N_BITS: u5 = 26;

    /// Element size in bytes
    pub const ELEMENT_SIZE: usize = 32;

    /// Total table memory requirement
    pub const TABLE_SIZE: usize = N * ELEMENT_SIZE; // ~2 GB

    /// Epoch length for table seed rotation
    pub const EPOCH_LENGTH: u32 = 1024;

    /// Compute the PoW hit value for a header
    /// Returns BigInt256 that must be < target (from nBits)
    ///
    /// Cost: ~700 JitCost (multiple Blake2b256 computations)
    pub fn powHit(
        header_without_pow: []const u8,
        nonce: u64,
        height: u32,
    ) BigInt256 {
        // Step 1: Compute message hash
        const msg = Blake2b256.hash(header_without_pow);

        // Step 2: Generate table seed from height epoch
        const seed = computeTableSeed(height);

        // Step 3: Compute k-sum of table elements
        var sum = UnsignedBigInt256.ZERO;
        var nonce_bytes: [8]u8 = undefined;
        std.mem.writeInt(u64, &nonce_bytes, nonce, .big);

        for (0..K) |i| {
            // Derive index from hash(msg || nonce || i)
            var index_input: [32 + 8 + 4]u8 = undefined;
            @memcpy(index_input[0..32], &msg);
            @memcpy(index_input[32..40], &nonce_bytes);
            std.mem.writeInt(u32, index_input[40..44], @intCast(i), .big);

            const index_hash = Blake2b256.hash(&index_input);
            const idx = std.mem.readInt(u32, index_hash[0..4], .big) % N;

            // Look up table element
            const element = computeTableElement(seed, idx);
            sum = addUnchecked(sum, element);
        }

        // Step 4: Final hash to get PoW hit
        var final_input: [32 + 8 + 32]u8 = undefined;
        @memcpy(final_input[0..32], &msg);
        @memcpy(final_input[32..40], &nonce_bytes);
        @memcpy(final_input[40..72], &sum.toBytes());

        const hit_hash = Blake2b256.hash(&final_input);
        return BigInt256.fromBytes(hit_hash);
    }

    /// Compute table seed from block height
    /// Seed changes every EPOCH_LENGTH blocks to prevent precomputation
    fn computeTableSeed(height: u32) [32]u8 {
        const epoch = height / EPOCH_LENGTH;
        var epoch_bytes: [4]u8 = undefined;
        std.mem.writeInt(u32, &epoch_bytes, epoch, .big);
        return Blake2b256.hash(&epoch_bytes);
    }

    /// Compute table element at given index
    /// This is the memory-hard part - miners must store or recompute
    fn computeTableElement(seed: [32]u8, idx: u32) UnsignedBigInt256 {
        // Element = H(seed || idx || 0) || H(seed || idx || 1) || ...
        // Combined to form 256-bit value
        var result: [32]u8 = undefined;

        for (0..4) |chunk| {
            var input: [32 + 4 + 1]u8 = undefined;
            @memcpy(input[0..32], &seed);
            std.mem.writeInt(u32, input[32..36], idx, .big);
            input[36] = @intCast(chunk);

            const chunk_hash = Blake2b256.hash(&input);
            @memcpy(result[chunk * 8 ..][0..8], chunk_hash[0..8]);
        }

        return UnsignedBigInt256.fromBytes(result);
    }
};
```

### Header.checkPow

```zig
/// Verify that a block header satisfies the PoW difficulty requirement
///
/// checkPow() returns true iff powHit(header) < decodeNBits(header.nBits)
///
/// Cost: FixedCost(700) - approximately 2×32 hash computations
pub fn checkPow(header: Header) bool {
    // Serialize header without PoW solution
    const header_bytes = header.serializeWithoutPow();

    // Compute PoW hit
    const hit = Autolykos2.powHit(
        header_bytes,
        header.powSolutions.n, // nonce
        header.height,
    );

    // Decode difficulty target from nBits
    const target = NBits.decode(header.nBits);

    // Valid if hit < target
    return hit.lessThan(target);
}
```

### Why Memory-Hard?

Autolykos2's memory requirement (~2GB) provides ASIC resistance:

1. **Table storage**: Miners must maintain the full table in fast memory
2. **Random access**: k=32 random lookups per attempt prevents caching tricks
3. **Epoch rotation**: Table changes every ~1024 blocks, invalidating precomputation
4. **GPU-friendly**: Memory bandwidth is the bottleneck, favoring commodity GPUs

## NBits Difficulty Encoding

The `nBits` field in block headers uses a compact encoding for the difficulty target[^11]:

```
NBits Format
═══════════════════════════════════════════════════════════════════

Format: 0xAABBBBBB (4 bytes)
  AA     = exponent (1 byte)
  BBBBBB = mantissa (3 bytes)

Value = mantissa × 256^(exponent - 3)

Example:
  nBits = 0x1d00ffff
  exponent = 0x1d = 29
  mantissa = 0x00ffff = 65535
  target = 65535 × 256^(29-3) = 65535 × 256^26
```

### Implementation

```zig
const NBits = struct {
    /// Decode nBits to BigInt target
    /// Cost: FixedCost(10)
    pub fn decode(nBits: i64) BigInt256 {
        const n = @as(u32, @intCast(nBits & 0xFFFFFFFF));
        const exp: u8 = @intCast((n >> 24) & 0xFF);
        const mantissa: u32 = n & 0x00FFFFFF;

        if (exp <= 3) {
            // Small exponent: right shift mantissa
            const shift = (3 - exp) * 8;
            return BigInt256.fromInt(mantissa >> @intCast(shift));
        } else {
            // Normal case: left shift mantissa
            const shift = (exp - 3) * 8;
            return BigInt256.fromInt(mantissa).shiftLeft(shift);
        }
    }

    /// Encode BigInt to nBits
    /// Cost: FixedCost(10)
    pub fn encode(target: BigInt256) i64 {
        // Find the byte length of target
        const bytes = target.toBytes();
        var byte_len: u8 = 32;
        for (bytes) |b| {
            if (b != 0) break;
            byte_len -= 1;
        }

        if (byte_len == 0) return 0;

        // Extract top 3 bytes as mantissa
        const start = 32 - byte_len;
        var mantissa: u32 = 0;

        if (byte_len >= 3) {
            mantissa = (@as(u32, bytes[start]) << 16) |
                       (@as(u32, bytes[start + 1]) << 8) |
                       @as(u32, bytes[start + 2]);
        } else if (byte_len == 2) {
            mantissa = (@as(u32, bytes[start]) << 8) |
                       @as(u32, bytes[start + 1]);
        } else {
            mantissa = bytes[start];
        }

        // Handle sign bit in mantissa (MSB must be 0)
        if (mantissa & 0x00800000 != 0) {
            mantissa >>= 8;
            byte_len += 1;
        }

        return @as(i64, byte_len) << 24 | @as(i64, mantissa);
    }
};
```

## Global Serialization Methods

v6 adds methods to `Global` for value serialization[^12]:

### serialize

```zig
/// Serialize any value to bytes using SigmaSerializer
/// Works for all serializable types
/// Cost: Varies by type complexity
pub fn serialize(comptime T: type, value: T) ![]const u8 {
    var buffer = std.ArrayList(u8).init(allocator);
    const writer = buffer.writer();

    try SigmaSerializer.serialize(T, value, writer);

    return buffer.toOwnedSlice();
}
```

### fromBigEndianBytes

```zig
/// Deserialize numeric type from big-endian bytes
/// Generic over numeric types
/// Cost: FixedCost(5) for primitives
pub fn fromBigEndianBytes(comptime T: type, bytes: []const u8) !T {
    const size = @sizeOf(T);
    if (bytes.len != size) return error.InvalidLength;

    var arr: [size]u8 = undefined;
    @memcpy(&arr, bytes);

    return std.mem.readInt(T, &arr, .big);
}
```

## Cost Model for v6 Operations

```
v6 Operation Costs
═══════════════════════════════════════════════════════════════════

Operation                    Cost Type       Value    Notes
─────────────────────────────────────────────────────────────────────
Modular Arithmetic:
  mod(a, m)                  Fixed           20       Division
  modInverse(a, m)           Fixed           150      Extended Euclid
  plusMod(a, b, m)           Fixed           30       Add + mod
  subtractMod(a, b, m)       Fixed           30       Sub + mod
  multiplyMod(a, b, m)       Fixed           40       Mul + mod

Bitwise (all types):
  bitwiseInverse(x)          Fixed           5        Single op
  bitwiseOr(x, y)            Fixed           5        Single op
  bitwiseAnd(x, y)           Fixed           5        Single op
  bitwiseXor(x, y)           Fixed           5        Single op
  shiftLeft(x, n)            Fixed           5        Single op
  shiftRight(x, n)           Fixed           5        Single op
  toBytes(x)                 Fixed           5        Conversion
  toBits(x)                  Fixed           5        Conversion

Collections:
  patch(xs, from, p, r)      PerItem(30,2,10)        O(n)
  updated(xs, idx, v)        PerItem(20,1,10)        O(n) copy
  updateMany(xs, is, vs)     PerItem(20,2,10)        O(n)
  reverse(xs)                PerItem (append)        O(n)
  startsWith(xs, p)          PerItem (zip)           O(|p|)
  endsWith(xs, s)            PerItem (zip)           O(|s|)
  get(xs, idx)               Fixed           14      O(1)

Cryptographic:
  expUnsigned(g, k)          Fixed           900     Scalar mult
  checkPow(header)           Fixed           700     ~32 hashes
  powHit(...)                Dynamic                 Autolykos2

Serialization:
  serialize(v)               Varies                  Type-dependent
  fromBigEndianBytes(b)      Fixed           5       Simple parse
  encodeNBits(n)             Fixed           10      Encoding
  decodeNBits(n)             Fixed           10      Decoding
```

## Migration Guide

### Version Checking in Scripts

```
// ErgoScript: Check v6 availability
val canUseV6 = getVar[Boolean](127).getOrElse(false)

// Conditional v6 feature usage
if (canUseV6) {
  // Use v6 features
  val x: UnsignedBigInt = ...
  val inv = x.modInverse(p)
} else {
  // Fallback for pre-v6
}
```

### When to Use v6 Features

| Feature | Use When |
|---------|----------|
| `SUnsignedBigInt` | Cryptographic protocols requiring modular arithmetic |
| `modInverse` | Computing multiplicative inverses in finite fields |
| Bitwise ops | Bit manipulation, flags, compact encodings |
| `patch/updated` | Immutable collection updates in contracts |
| `get` | Safe array access without exceptions |
| `checkPow` | On-chain PoW verification for sidechains/merged mining |

### Backward Compatibility

- v6 features are **only available** when `VersionContext.isV6Activated()` returns true
- Scripts using v6 features will fail validation on pre-v6 nodes
- Design scripts with fallback paths for pre-v6 compatibility during transition

## Summary

This chapter covered the v6 protocol features that expand ErgoTree's capabilities:

- **SUnsignedBigInt** provides 256-bit unsigned integers for cryptographic modular arithmetic, with six new methods (mod, modInverse, plusMod, subtractMod, multiplyMod, toSigned)

- **Bitwise operations** (AND, OR, XOR, NOT, shifts) are now available on all numeric types with consistent semantics and low cost (5 JitCost each)

- **Collection methods** (patch, updated, updateMany, reverse, startsWith, endsWith, get) enable efficient immutable collection manipulation

- **Autolykos2 PoW** verification is exposed through `Header.checkPow()` and `Global.powHit()`, enabling on-chain proof-of-work validation

- **NBits encoding** provides compact difficulty target representation with `encodeNBits`/`decodeNBits`

- **Serialization** methods (`Global.serialize`, `fromBigEndianBytes`) support arbitrary value serialization

- **Cost model** assigns appropriate costs to each operation, with modInverse (150) and checkPow (700) being the most expensive due to their computational complexity

---

*[Previous: Chapter 31](./ch31-performance-engineering.md) | [Next: Appendix A](../appendices/appendix-a-type-codes.md)*

[^1]: Scala: [`CUnsignedBigInt.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/CUnsignedBigInt.scala)

[^2]: Rust: [`unsignedbigint256.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/unsignedbigint256.rs)

[^3]: Scala: [`methods.scala:570-625`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala#L570-L625) (SUnsignedBigIntMethods)

[^4]: Rust: [`snumeric.rs:381-491`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/snumeric.rs#L381-L491)

[^5]: Scala: [`methods.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala) (Bitwise method definitions)

[^6]: Rust: [`snumeric.rs:73-264`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval/snumeric.rs#L73-L264)

[^7]: Scala: [`Colls.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/Colls.scala) (Collection trait)

[^8]: Rust: [`scoll.rs:140-266`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/types/scoll.rs#L140-L266)

[^9]: Ergo: [Autolykos PoW](https://docs.ergoplatform.com/mining/autolykos/)

[^10]: Rust: [`Header` type](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/chain/header.rs)

[^11]: Bitcoin Wiki: [Difficulty](https://en.bitcoin.it/wiki/Difficulty)

[^12]: Scala: [`sglobal.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/methods.scala) (SGlobalMethods)
