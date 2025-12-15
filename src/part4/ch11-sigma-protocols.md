# Chapter 11: Sigma Protocols

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Elliptic curve cryptography ([Chapter 9](./ch09-elliptic-curve-cryptography.md))
- Hash functions ([Chapter 10](./ch10-hash-functions.md))
- Discrete logarithm problem

## Learning Objectives

- Understand three-move Sigma protocol structure
- Implement Schnorr (DLog) and Diffie-Hellman Tuple protocols
- Master protocol composition (AND, OR, THRESHOLD)
- Apply Fiat-Shamir transformation for non-interactive proofs

## Sigma Protocol Structure

A Sigma protocol is a three-move interactive proof[^1][^2]:

```
Sigma Protocol Flow
─────────────────────────────────────────────────────

    Prover (P)                           Verifier (V)
        │                                      │
        │   ────────  a (commitment) ───────>  │
        │                                      │
        │   <───────  e (challenge) ─────────  │
        │                                      │
        │   ────────  z (response) ─────────>  │
        │                                      │
        │                          Verify(a, e, z)?
```

### Message Types

```zig
/// First message: prover's commitment
const FirstProverMessage = union(enum) {
    dlog: FirstDlogProverMessage,
    dht: FirstDhtProverMessage,

    pub fn bytes(self: FirstProverMessage) []const u8 {
        return switch (self) {
            .dlog => |m| m.a.serialize(),
            .dht => |m| m.a.serialize() ++ m.b.serialize(),
        };
    }
};

/// Second message: prover's response
const SecondProverMessage = union(enum) {
    dlog: SecondDlogProverMessage,
    dht: SecondDhtProverMessage,
};

/// Challenge from verifier (192 bits = 24 bytes)
const Challenge = [FiatShamir.SOUNDNESS_BYTES]u8;
```

## Schnorr Protocol (Discrete Log)

Proves knowledge of secret `w` such that `h = g^w`[^3][^4]:

```
Schnorr Protocol Steps
─────────────────────────────────────────────────────
Given: g (generator), h = g^w (public key), w (secret)

Step     Message    Computation
─────────────────────────────────────────────────────
1. Commit    a      r ← random, a = g^r
2. Challenge e      Verifier sends random e
3. Response  z      z = r + e·w (mod q)
4. Verify    ✓      g^z = a · h^e
```

### Implementation

```zig
const DlogProverInput = struct {
    /// Secret scalar w in [0, q-1]
    w: Scalar,

    /// Compute public image h = g^w
    pub fn publicImage(self: *const DlogProverInput) ProveDlog {
        const g = DlogGroup.generator();
        const h = DlogGroup.exponentiate(&g, &self.w);
        return ProveDlog{ .h = h };
    }

    /// Generate random secret
    pub fn random(rng: std.rand.Random) DlogProverInput {
        return .{ .w = Scalar.random(rng) };
    }
};

/// First message: commitment a = g^r
const FirstDlogProverMessage = struct {
    a: EcPoint,

    pub fn bytes(self: *const FirstDlogProverMessage) [33]u8 {
        return GroupElementSerializer.serialize(&self.a);
    }
};

/// Second message: response z
const SecondDlogProverMessage = struct {
    z: Scalar,
};
```

### Prover Steps

```zig
const DlogProver = struct {
    /// Step 1: Generate commitment (real proof)
    pub fn firstMessage(rng: std.rand.Random) struct { r: Scalar, msg: FirstDlogProverMessage } {
        const r = Scalar.random(rng);
        const g = DlogGroup.generator();
        const a = DlogGroup.exponentiate(&g, &r);
        return .{ .r = r, .msg = .{ .a = a } };
    }

    /// Step 3: Compute response z = r + e·w (mod q)
    pub fn secondMessage(
        private_input: *const DlogProverInput,
        r: Scalar,
        challenge: *const Challenge,
    ) SecondDlogProverMessage {
        const e = Scalar.fromBytes(challenge);
        const ew = e.mul(&private_input.w);  // e * w mod q
        const z = r.add(&ew);                 // r + ew mod q
        return .{ .z = z };
    }
};
```

### Simulation

For OR composition, we need to simulate proofs without knowing the secret[^5]:

```zig
/// Simulate transcript without knowing secret
/// Given challenge e, produce valid-looking (a, z)
pub fn simulate(
    public_input: *const ProveDlog,
    challenge: *const Challenge,
    rng: std.rand.Random,
) struct { first: FirstDlogProverMessage, second: SecondDlogProverMessage } {
    // SAMPLE random z
    const z = Scalar.random(rng);

    // COMPUTE a = g^z · h^(-e)
    // This satisfies verification equation: g^z = a · h^e
    const e = Scalar.fromBytes(challenge);
    const minus_e = e.negate();
    const g = DlogGroup.generator();
    const h = public_input.h;

    const g_to_z = DlogGroup.exponentiate(&g, &z);
    const h_to_minus_e = DlogGroup.exponentiate(&h, &minus_e);
    const a = DlogGroup.multiply(&g_to_z, &h_to_minus_e);

    return .{
        .first = .{ .a = a },
        .second = .{ .z = z },
    };
}
```

### Verification (Commitment Reconstruction)

```zig
/// Verify: reconstruct a from z and e, check equality
/// g^z = a · h^e  =>  a = g^z / h^e
pub fn computeCommitment(
    proposition: *const ProveDlog,
    challenge: *const Challenge,
    second_msg: *const SecondDlogProverMessage,
) EcPoint {
    const g = DlogGroup.generator();
    const h = proposition.h;
    const e = Scalar.fromBytes(challenge);

    const g_to_z = DlogGroup.exponentiate(&g, &second_msg.z);
    const h_to_e = DlogGroup.exponentiate(&h, &e);
    const h_to_e_inv = DlogGroup.inverse(&h_to_e);

    return DlogGroup.multiply(&g_to_z, &h_to_e_inv);
}
```

## Diffie-Hellman Tuple Protocol

Proves knowledge of `w` such that `u = g^w` AND `v = h^w`[^6][^7]:

```
DHT Protocol: Prove (u, v) share the same discrete log

Given: g, h (generators), u = g^w, v = h^w (public tuple)

Step     Message      Computation
─────────────────────────────────────────────────────
1. Commit    (a, b)    r ← random, a = g^r, b = h^r
2. Challenge e         Verifier sends random e
3. Response  z         z = r + e·w (mod q)
4. Verify    ✓         g^z = a·u^e  AND  h^z = b·v^e
```

### Implementation

```zig
const ProveDhTuple = struct {
    g: EcPoint,
    h: EcPoint,
    u: EcPoint,  // u = g^w
    v: EcPoint,  // v = h^w

    pub const OP_CODE = OpCode.ProveDiffieHellmanTuple;
};

const FirstDhtProverMessage = struct {
    a: EcPoint,  // a = g^r
    b: EcPoint,  // b = h^r

    pub fn bytes(self: *const FirstDhtProverMessage) [66]u8 {
        var result: [66]u8 = undefined;
        @memcpy(result[0..33], &GroupElementSerializer.serialize(&self.a));
        @memcpy(result[33..66], &GroupElementSerializer.serialize(&self.b));
        return result;
    }
};

const DhtProver = struct {
    pub fn firstMessage(
        public_input: *const ProveDhTuple,
        rng: std.rand.Random,
    ) struct { r: Scalar, msg: FirstDhtProverMessage } {
        const r = Scalar.random(rng);
        const a = DlogGroup.exponentiate(&public_input.g, &r);
        const b = DlogGroup.exponentiate(&public_input.h, &r);
        return .{ .r = r, .msg = .{ .a = a, .b = b } };
    }
};
```

## SigmaBoolean Proposition Types

Propositions form a tree structure[^8][^9]:

```zig
const SigmaBoolean = union(enum) {
    /// Leaf: prove knowledge of discrete log
    prove_dlog: ProveDlog,
    /// Leaf: prove DHT equality
    prove_dh_tuple: ProveDhTuple,
    /// Conjunction: all children must be proven
    cand: Cand,
    /// Disjunction: at least one child proven
    cor: Cor,
    /// Threshold: k-of-n children proven
    cthreshold: Cthreshold,
    /// Trivially true
    trivial_true: void,
    /// Trivially false
    trivial_false: void,

    pub fn opCode(self: SigmaBoolean) OpCode {
        return switch (self) {
            .prove_dlog => OpCode.ProveDlog,
            .prove_dh_tuple => OpCode.ProveDiffieHellmanTuple,
            .cand => OpCode.SigmaAnd,
            .cor => OpCode.SigmaOr,
            .cthreshold => OpCode.Atleast,
            else => OpCode.Constant,
        };
    }

    /// Count nodes in proposition tree
    pub fn size(self: SigmaBoolean) usize {
        return switch (self) {
            .cand => |c| 1 + sumChildSizes(c.children),
            .cor => |c| 1 + sumChildSizes(c.children),
            .cthreshold => |c| 1 + sumChildSizes(c.children),
            else => 1,
        };
    }
};

const ProveDlog = struct {
    h: EcPoint,  // Public key h = g^w

    pub const OP_CODE = OpCode.ProveDlog;
};

const Cand = struct {
    children: []const SigmaBoolean,
};

const Cor = struct {
    children: []const SigmaBoolean,
};

const Cthreshold = struct {
    k: u8,  // Threshold
    children: []const SigmaBoolean,
};
```

## Protocol Composition

### AND Composition

All children share the same challenge[^10]:

```
       Challenge e
          │
    ┌─────┴─────┐
    │           │
  σ₁(e)       σ₂(e)
   real        real
```

```zig
/// AND: prove all children with same challenge
fn proveAnd(
    children: []const *SigmaBoolean,
    secrets: []const PrivateInput,
    challenge: *const Challenge,
) []const ProofNode {
    var proofs = allocator.alloc(ProofNode, children.len);
    for (children, secrets, 0..) |child, secret, i| {
        proofs[i] = proveReal(child, secret, challenge);
    }
    return proofs;
}
```

### OR Composition

At least one child is real; others are simulated[^11]:

```
       Challenge e
          │
    ┌─────┴─────┐
    │           │
  σ₁(e₁)      σ₂(e₂)
   REAL       SIMULATED

Constraint: e₁ ⊕ e₂ = e (XOR)
```

```zig
/// OR: one real proof, rest simulated
/// Challenges must XOR to root challenge
fn proveOr(
    children: []const *SigmaBoolean,
    real_index: usize,
    secret: PrivateInput,
    challenge: *const Challenge,
    rng: std.rand.Random,
) []const ProofNode {
    var proofs = allocator.alloc(ProofNode, children.len);
    var challenge_sum: Challenge = [_]u8{0} ** FiatShamir.SOUNDNESS_BYTES;

    // First: generate simulated proofs with random challenges
    for (children, 0..) |child, i| {
        if (i != real_index) {
            var sim_challenge: Challenge = undefined;
            rng.bytes(&sim_challenge);
            proofs[i] = simulate(child, &sim_challenge, rng);
            xorChallenge(&challenge_sum, &sim_challenge);
        }
    }

    // Derive real challenge: e_real = e ⊕ (sum of simulated challenges)
    var real_challenge: Challenge = undefined;
    for (0..FiatShamir.SOUNDNESS_BYTES) |i| {
        real_challenge[i] = challenge[i] ^ challenge_sum[i];
    }

    proofs[real_index] = proveReal(children[real_index], secret, &real_challenge);
    return proofs;
}
```

### THRESHOLD (k-of-n)

Uses polynomial interpolation over GF(2^192)[^12]:

```
Threshold k-of-n Challenge Distribution
─────────────────────────────────────────────────────
- Construct polynomial p(x) of degree k-1
- p(0) = e (root challenge)
- Each child i gets challenge p(i)
- k real children, (n-k) simulated
```

```zig
const GF2_192 = struct {
    /// 192 bits = 3 × 64-bit words
    words: [3]u64,

    pub fn fromChallenge(c: *const Challenge) GF2_192 {
        var self = GF2_192{ .words = .{ 0, 0, 0 } };
        // Load 24 bytes into 3 words (only 192 bits used)
        @memcpy(std.mem.asBytes(&self.words[0])[0..8], c[0..8]);
        @memcpy(std.mem.asBytes(&self.words[1])[0..8], c[8..16]);
        @memcpy(std.mem.asBytes(&self.words[2])[0..8], c[16..24]);
        return self;
    }

    pub fn add(a: *const GF2_192, b: *const GF2_192) GF2_192 {
        // Addition in GF(2^192) is XOR
        return .{ .words = .{
            a.words[0] ^ b.words[0],
            a.words[1] ^ b.words[1],
            a.words[2] ^ b.words[2],
        } };
    }

    // Multiplication uses polynomial representation with reduction
    pub fn mul(a: *const GF2_192, b: *const GF2_192) GF2_192 {
        // ... polynomial multiplication mod irreducible
        _ = a;
        _ = b;
        return .{ .words = .{ 0, 0, 0 } };
    }
};

const GF2_192_Poly = struct {
    coefficients: []GF2_192,
    degree: usize,

    /// Evaluate polynomial at point x using Horner's method
    pub fn evaluate(self: *const GF2_192_Poly, x: u8) GF2_192 {
        var result = self.coefficients[self.degree];
        var i = self.degree;
        while (i > 0) {
            i -= 1;
            result = GF2_192.add(&GF2_192.mulByByte(&result, x), &self.coefficients[i]);
        }
        return result;
    }

    /// Lagrange interpolation through given points
    pub fn interpolate(
        points: []const u8,
        values: []const GF2_192,
        value_at_0: GF2_192,
    ) GF2_192_Poly {
        // Construct unique polynomial of degree (n-1)
        // passing through n points with p(0) = value_at_0
        _ = points;
        _ = values;
        _ = value_at_0;
        return undefined;
    }
};
```

## Proof Trees

Track proof state during proving[^13]:

```zig
const UnprovenTree = union(enum) {
    leaf: UnprovenLeaf,
    conjecture: UnprovenConjecture,
};

const UnprovenLeaf = struct {
    proposition: SigmaBoolean,
    position: NodePosition,
    simulated: bool,
    commitment_opt: ?FirstProverMessage,
    randomness_opt: ?Scalar,
    challenge_opt: ?Challenge,
};

const UnprovenConjecture = struct {
    conj_type: enum { and, or_, threshold },
    children: []UnprovenTree,
    position: NodePosition,
    simulated: bool,
    challenge_opt: ?Challenge,
    k: ?u8,  // For threshold
    polynomial_opt: ?GF2_192_Poly,
};

/// Position in tree: "0-2-1" means root → child 2 → child 1
const NodePosition = struct {
    positions: []const usize,

    pub fn child(self: NodePosition, idx: usize) NodePosition {
        return .{ .positions = self.positions ++ &[_]usize{idx} };
    }

    pub const CRYPTO_PREFIX = NodePosition{ .positions = &.{0} };
};
```

## Fiat-Shamir Transformation

Convert interactive to non-interactive by deriving challenge from hash[^14]:

```zig
/// Derive challenge from tree serialization
pub fn fiatShamirChallenge(tree: *const ProofTree) Challenge {
    var buf = std.ArrayList(u8).init(allocator);
    fiatShamirSerialize(tree, &buf);
    return FiatShamir.hashFn(buf.items);
}

fn fiatShamirSerialize(tree: *const ProofTree, writer: anytype) !void {
    const INTERNAL_PREFIX: u8 = 0;
    const LEAF_PREFIX: u8 = 1;

    switch (tree.*) {
        .leaf => |leaf| {
            try writer.writeByte(LEAF_PREFIX);
            // Proposition bytes
            const prop_bytes = try leaf.proposition.toErgoTreeBytes();
            try writer.writeIntBig(i16, @intCast(prop_bytes.len));
            try writer.writeAll(prop_bytes);
            // Commitment bytes
            const commitment = leaf.commitment_opt orelse return error.NoCommitment;
            const comm_bytes = commitment.bytes();
            try writer.writeIntBig(i16, @intCast(comm_bytes.len));
            try writer.writeAll(comm_bytes);
        },
        .conjecture => |conj| {
            try writer.writeByte(INTERNAL_PREFIX);
            try writer.writeByte(@intFromEnum(conj.conj_type));
            if (conj.k) |k| try writer.writeByte(k);
            try writer.writeIntBig(i16, @intCast(conj.children.len));
            for (conj.children) |child| {
                try fiatShamirSerialize(child, writer);
            }
        },
    }
}
```

## Security Properties

```
Security Properties
─────────────────────────────────────────────────────
Property         Meaning
─────────────────────────────────────────────────────
Completeness     Honest prover always convinces
Soundness        Cheater succeeds with prob ≤ 2^-192
Zero-Knowledge   Proof reveals nothing about secret
Special Sound.   Two transcripts extract secret
```

## Summary

- **Sigma protocols** are three-move (commitment, challenge, response) proofs
- **Schnorr** proves knowledge of discrete log: `h = g^w`
- **DHT** proves equality of discrete logs across two bases
- **AND** uses same challenge for all children
- **OR** distributes challenges via XOR constraint
- **THRESHOLD** uses GF(2^192) polynomial interpolation
- **Simulation** enables OR/threshold without knowing all secrets
- **Fiat-Shamir** derives challenge from hash of commitments

---

*Next: [Chapter 12: Evaluation Model](../part5/ch12-evaluation-model.md)*

[^1]: Scala: `interpreter/shared/src/main/scala/sigmastate/crypto/SigmaProtocolFunctions.scala`

[^2]: Rust: `ergotree-interpreter/src/sigma_protocol.rs`

[^3]: Scala: `interpreter/shared/src/main/scala/sigmastate/crypto/DLogProtocol.scala`

[^4]: Rust: `ergotree-interpreter/src/sigma_protocol/dlog_protocol.rs:10-47`

[^5]: Rust: `ergotree-interpreter/src/sigma_protocol/dlog_protocol.rs:73-93`

[^6]: Scala: `interpreter/shared/src/main/scala/sigmastate/crypto/DiffieHellmanTupleProtocol.scala`

[^7]: Rust: `ergotree-interpreter/src/sigma_protocol/dht_protocol.rs`

[^8]: Scala: `core/shared/src/main/scala/sigma/data/SigmaBoolean.scala`

[^9]: Rust: `ergotree-ir/src/sigma_protocol/sigma_boolean.rs:31-96`

[^10]: Rust: `ergotree-ir/src/sigma_protocol/sigma_boolean/cand.rs`

[^11]: Rust: `ergotree-ir/src/sigma_protocol/sigma_boolean/cor.rs`

[^12]: Scala: `interpreter/shared/src/main/scala/sigmastate/crypto/GF2_192_Poly.scala`

[^13]: Rust: `ergotree-interpreter/src/sigma_protocol/unproven_tree.rs`

[^14]: Rust: `ergotree-interpreter/src/sigma_protocol/fiat_shamir.rs`