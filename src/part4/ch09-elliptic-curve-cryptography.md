# Chapter 9: Elliptic Curve Cryptography

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Basic finite field and modular arithmetic
- Public key cryptography concepts
- Prior chapters: Type system ([Chapter 2](../part1/ch02-type-system.md))

## Learning Objectives

- Understand secp256k1 elliptic curve used in Sigma protocols
- Implement group operations in Zig
- Encode and decode group elements
- Work with multiplicative vs additive group notation

## The Secp256k1 Curve

Sigma protocols use **secp256k1**—the same curve as Bitcoin and Ethereum[^1][^2].

### Curve Definition

The curve is defined by:

```
y² = x³ + 7  (mod p)

where:
  p = 2²⁵⁶ - 2³² - 977  (field characteristic)
  n = group order        (number of points)
  G = generator point    (base point)
```

### Cryptographic Constants

```zig
const CryptoConstants = struct {
    /// Encoded group element size in bytes (compressed)
    pub const ENCODED_GROUP_ELEMENT_LENGTH: usize = 33;

    /// Group size in bits
    pub const GROUP_SIZE_BITS: u32 = 256;

    /// Challenge size for Sigma protocols
    /// Must be < GROUP_SIZE_BITS for security
    pub const SOUNDNESS_BITS: u32 = 192;

    /// Group order (number of curve points)
    /// n = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
    pub const GROUP_ORDER: [32]u8 = .{
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFE,
        0xBA, 0xAE, 0xDC, 0xE6, 0xAF, 0x48, 0xA0, 0x3B,
        0xBF, 0xD2, 0x5E, 0x8C, 0xD0, 0x36, 0x41, 0x41,
    };

    /// Field characteristic
    /// p = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
    pub const FIELD_PRIME: [32]u8 = .{
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xFF, 0xFF, 0xFF, 0xFE, 0xFF, 0xFF, 0xFC, 0x2F,
    };

    comptime {
        // Security constraint: 2^soundnessBits < groupOrder
        std.debug.assert(SOUNDNESS_BITS < GROUP_SIZE_BITS);
    }
};
```

## Group Element Representation

### EcPoint Structure

```zig
const EcPoint = struct {
    /// Compressed encoding size
    pub const GROUP_SIZE: usize = 33;

    /// Internal representation (projective coordinates)
    x: FieldElement,
    y: FieldElement,
    z: FieldElement,

    /// Identity element (point at infinity)
    pub const IDENTITY = EcPoint{
        .x = FieldElement.zero(),
        .y = FieldElement.one(),
        .z = FieldElement.zero(),
    };

    /// Generator point G
    pub const GENERATOR = init: {
        // secp256k1 generator coordinates
        const gx = FieldElement.fromHex(
            "79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798"
        );
        const gy = FieldElement.fromHex(
            "483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8"
        );
        break :init EcPoint{ .x = gx, .y = gy, .z = FieldElement.one() };
    };

    /// Check if this is the identity (infinity) point
    pub fn isIdentity(self: *const EcPoint) bool {
        return self.z.isZero();
    }

    /// Convert to affine coordinates
    pub fn toAffine(self: *const EcPoint) struct { x: FieldElement, y: FieldElement } {
        if (self.isIdentity()) return .{ .x = .zero(), .y = .zero() };
        const z_inv = self.z.inverse();
        return .{
            .x = self.x.mul(z_inv),
            .y = self.y.mul(z_inv),
        };
    }
};
```

## Group Operations

The discrete logarithm group interface provides standard operations[^3][^4]:

```
Operation           Notation      Description
─────────────────────────────────────────────────────
Exponentiate        g^x           Scalar multiplication
Multiply            g * h         Point addition
Inverse             g^(-1)        Point negation
Identity            1             Point at infinity
Generator           g             Base point G
```

### Group Interface

```zig
const DlogGroup = struct {
    /// The generator point
    pub fn generator() EcPoint {
        return EcPoint.GENERATOR;
    }

    /// The identity element (point at infinity)
    pub fn identity() EcPoint {
        return EcPoint.IDENTITY;
    }

    /// Check if point is identity
    pub fn isIdentity(point: *const EcPoint) bool {
        return point.isIdentity();
    }

    /// Exponentiate: base^exponent (scalar multiplication)
    pub fn exponentiate(base: *const EcPoint, exponent: *const Scalar) EcPoint {
        if (base.isIdentity()) return base.*;

        // Handle negative exponents
        var exp = exponent.*;
        if (exp.isNegative()) {
            exp = exp.mod(CryptoConstants.GROUP_ORDER);
        }

        return scalarMul(base, &exp);
    }

    /// Multiply two group elements: g1 * g2 (point addition)
    pub fn multiply(g1: *const EcPoint, g2: *const EcPoint) EcPoint {
        return pointAdd(g1, g2);
    }

    /// Compute inverse: g^(-1) (point negation)
    pub fn inverse(point: *const EcPoint) EcPoint {
        return EcPoint{
            .x = point.x,
            .y = point.y.negate(),
            .z = point.z,
        };
    }

    /// Create random group element
    pub fn randomElement(rng: std.rand.Random) EcPoint {
        const scalar = Scalar.random(rng);
        return exponentiate(&EcPoint.GENERATOR, &scalar);
    }
};
```

### Notation Translation

Sigma protocols use multiplicative notation while underlying libraries often use additive[^5]:

```
Sigma (multiplicative)    Library (additive)     Operation
──────────────────────────────────────────────────────────
g * h                     g + h                  Point addition
g^n                       n * g                  Scalar multiplication
g^(-1)                    -g                     Point negation
1 (identity)              O (origin)             Point at infinity
```

```zig
/// Wrapper translating multiplicative to additive notation
const MultiplicativeGroup = struct {
    /// Multiply in multiplicative notation = Add in additive
    pub fn mul(a: *const EcPoint, b: *const EcPoint) EcPoint {
        return pointAdd(a, b);
    }

    /// Exponentiate in multiplicative = Scalar multiply in additive
    pub fn exp(base: *const EcPoint, scalar: *const Scalar) EcPoint {
        return scalarMul(base, scalar);
    }

    /// Inverse in multiplicative = Negate in additive
    pub fn inv(p: *const EcPoint) EcPoint {
        return pointNegate(p);
    }
};
```

## Point Encoding

Group elements use compressed SEC1 encoding (33 bytes)[^6][^7]:

```
Compressed Point Format (33 bytes)
────────────────────────────────────────────────────

┌──────────┬────────────────────────────────────────┐
│ Byte 0   │           Bytes 1-32                   │
├──────────┼────────────────────────────────────────┤
│ 0x02     │    X coordinate (32 bytes, big-end)    │  Y is even
│ 0x03     │    X coordinate (32 bytes, big-end)    │  Y is odd
│ 0x00     │    31 zero bytes                       │  Identity
└──────────┴────────────────────────────────────────┘
```

### Serialization Implementation

```zig
const GroupElementSerializer = struct {
    const ENCODING_SIZE: usize = 33;

    /// Identity encoding (33 zero bytes)
    const IDENTITY_ENCODING = [_]u8{0} ** ENCODING_SIZE;

    pub fn serialize(point: *const EcPoint, writer: anytype) !void {
        if (point.isIdentity()) {
            try writer.writeAll(&IDENTITY_ENCODING);
            return;
        }

        const affine = point.toAffine();

        // Determine sign byte from Y coordinate parity
        const y_bytes = affine.y.toBytes();
        const sign_byte: u8 = if (y_bytes[31] & 1 == 0) 0x02 else 0x03;

        // Write sign byte + X coordinate
        try writer.writeByte(sign_byte);
        try writer.writeAll(&affine.x.toBytes());
    }

    pub fn deserialize(reader: anytype) !EcPoint {
        var buf: [ENCODING_SIZE]u8 = undefined;
        try reader.readNoEof(&buf);

        if (buf[0] == 0) {
            // Check all zeros for identity
            for (buf[1..]) |b| {
                if (b != 0) return error.InvalidEncoding;
            }
            return EcPoint.IDENTITY;
        }

        if (buf[0] != 0x02 and buf[0] != 0x03) {
            return error.InvalidPrefix;
        }

        // Recover Y from X using curve equation: y² = x³ + 7
        const x = FieldElement.fromBytes(buf[1..33]);
        const y_squared = x.cube().add(FieldElement.fromInt(7));
        var y = y_squared.sqrt() orelse return error.NotOnCurve;

        // Choose correct Y based on sign byte
        const y_is_odd = y.toBytes()[31] & 1 == 1;
        if ((buf[0] == 0x02) == y_is_odd) {
            y = y.negate();
        }

        const point = EcPoint{ .x = x, .y = y, .z = FieldElement.one() };

        // CRITICAL: Validate point is on curve and in correct subgroup
        // This prevents invalid curve attacks. See ZIGMA_STYLE.md.
        // if (!point.isOnCurve()) return error.NotOnCurve;
        // if (!point.isInSubgroup()) return error.InvalidSubgroup;

        return point;
    }
};
```

### Why Compressed Encoding?

```
Format          Size     Content
────────────────────────────────────────────────────
Compressed      33 B     Sign (1) + X (32)
Uncompressed    65 B     0x04 (1) + X (32) + Y (32)
Savings         49%      Y recovered from curve equation
```

## Coordinate Systems

### Affine vs Projective

Libraries use projective coordinates internally for efficiency:

```
Coordinate System    Representation    Division Required
──────────────────────────────────────────────────────────
Affine               (x, y)            Per operation
Projective           (X, Y, Z)         Only at end
                     x = X/Z
                     y = Y/Z
```

### Normalization

```zig
/// Normalize point to affine coordinates
/// Required before: encoding, comparison, coordinate access
pub fn normalize(point: *const EcPoint) EcPoint {
    if (point.isIdentity()) return point.*;

    const z_inv = point.z.inverse();
    const z_inv_sq = z_inv.square();
    const z_inv_cu = z_inv_sq.mul(z_inv);

    return EcPoint{
        .x = point.x.mul(z_inv_sq),
        .y = point.y.mul(z_inv_cu),
        .z = FieldElement.one(),
    };
}
```

## Random Scalar Generation

Secure random scalars for key generation[^8]:

```zig
const Scalar = struct {
    bytes: [32]u8,

    /// Generate random scalar in [1, n-1] where n is group order
    pub fn random(rng: std.rand.Random) Scalar {
        while (true) {
            var bytes: [32]u8 = undefined;
            rng.bytes(&bytes);

            // Ensure scalar < group order
            if (lessThan(&bytes, &CryptoConstants.GROUP_ORDER)) {
                // Ensure non-zero
                var is_zero = true;
                for (bytes) |b| {
                    if (b != 0) { is_zero = false; break; }
                }
                if (!is_zero) {
                    return Scalar{ .bytes = bytes };
                }
            }
        }
    }

    /// Constant-time comparison to prevent timing attacks
    fn lessThan(a: *const [32]u8, b: *const [32]u8) bool {
        // NOTE: This simplified version is NOT constant-time.
        // In production, use constant-time comparison like:
        //   var borrow: u1 = 0;
        //   for (a.*, b.*) |ai, bi| {
        //       borrow = @intFromBool(ai < bi) | (borrow & @intFromBool(ai == bi));
        //   }
        //   return borrow == 1;
        // See ZIGMA_STYLE.md for constant-time crypto requirements.
        for (a.*, b.*) |ai, bi| {
            if (ai < bi) return true;
            if (ai > bi) return false;
        }
        return false;
    }
};
```

## Security Properties

### Discrete Logarithm Assumption

The security relies on the hardness of the DLP[^9]:

```
Given:  g (generator), h = g^x (public key)
Find:   x (secret key)

Best known attack: ~2^128 operations for secp256k1
```

### Soundness Parameter

The `SOUNDNESS_BITS = 192` determines:
- Challenge size in Sigma protocols
- Security level against malicious provers
- Constraint: `2^192 < n` (group order)

```zig
comptime {
    // Verify soundness constraint
    // 2^soundnessBits must be less than group order
    // Group order ≈ 2^256, so 192 < 256 satisfies this
    std.debug.assert(CryptoConstants.SOUNDNESS_BITS < 256);
}
```

## Summary

- **secp256k1** provides the elliptic curve foundation for all Sigma cryptography
- **Group elements** are 33 bytes (compressed SEC1 encoding)
- **Multiplicative notation** (exponentiate, multiply) maps to additive operations
- **SOUNDNESS_BITS = 192** is critical for protocol security
- **DlogGroup** interface: exponentiate, multiply, inverse, identity
- **Projective coordinates** optimize computation; normalize for encoding/comparison

---

*Next: [Chapter 10: Hash Functions](./ch10-hash-functions.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/crypto/CryptoConstants.scala`

[^2]: Rust: `ergo-chain-types/src/ec_point.rs:41-51`

[^3]: Scala: `data/shared/src/main/scala/sigma/crypto/DlogGroup.scala`

[^4]: Rust: `ergotree-ir/src/sigma_protocol/dlog_group.rs:39-84`

[^5]: Scala: `core/jvm/src/main/scala/sigma/crypto/Platform.scala:217-225`

[^6]: Scala: `core/shared/src/main/scala/sigma/serialization/GroupElementSerializer.scala`

[^7]: Rust: `ergo-chain-types/src/ec_point.rs:120-146`

[^8]: Rust: `ergotree-ir/src/sigma_protocol/dlog_group.rs:40-43`

[^9]: Scala: `data/shared/src/main/scala/sigma/crypto/CryptoConstants.scala:70-75`