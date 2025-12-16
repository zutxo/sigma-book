# Chapter 1: Introduction to Sigma and ErgoTree

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Basic blockchain and UTXO model knowledge
- Familiarity with any systems programming language
- Public key cryptography fundamentals (public keys, signatures)

## Learning Objectives

- Understand ErgoScript, ErgoTree, and SigmaBoolean relationships
- Describe the UTXO model and script-guarded spending
- Explain the prover/verifier separation
- Identify SigmaState interpreter components

## What is Sigma?

**Sigma** (Σ) refers to **Σ-protocols**—cryptographic protocols enabling zero-knowledge proofs of secret knowledge[^1]. The name reflects their three-move structure:

1. **Commitment**: Prover sends a commitment
2. **Challenge**: Verifier sends a random challenge
3. **Response**: Prover sends a response proving knowledge

## The Three Layers

```
┌─────────────────────────────────────┐
│           ErgoScript                │  High-level language
│     (Human-readable source)         │
└─────────────────┬───────────────────┘
                  │ Compilation
                  ▼
┌─────────────────────────────────────┐
│            ErgoTree                 │  Intermediate representation
│      (Typed AST / Bytecode)         │  (Serialized in UTXOs)
└─────────────────┬───────────────────┘
                  │ Evaluation
                  ▼
┌─────────────────────────────────────┐
│          SigmaBoolean               │  Cryptographic proposition
│      (Sigma protocol tree)          │  (What needs to be proven)
└─────────────────────────────────────┘
```

### ErgoScript

High-level, statically-typed language with Scala-like syntax[^2]:

- First-class lambdas and higher-order functions
- Call-by-value evaluation
- Local type inference
- Blocks as expressions

```zig
// Zig representation of an ErgoScript contract
const Contract = struct {
    freeze_deadline: i32,
    pk_owner: SigmaProp,

    pub fn evaluate(self: Contract, height: i32) SigmaProp {
        const deadline_passed = height > self.freeze_deadline;
        return SigmaProp.and(
            SigmaProp.fromBool(deadline_passed),
            self.pk_owner,
        );
    }
};
```

### ErgoTree

Compiled bytecode representation stored on-chain[^3][^4]:

- Typed abstract syntax tree (AST)
- Serialized as bytes in UTXOs
- Deterministically interpretable
- Version-controlled for soft-fork upgrades

```zig
const ErgoTree = struct {
    header: HeaderType,
    constants: []const Constant,
    root: union(enum) {
        parsed: SigmaPropValue,
        unparsed: UnparsedTree,
    },

    /// Header byte layout:
    /// Bit 7: Multi-byte header flag
    /// Bit 6: Reserved (GZIP)
    /// Bit 5: Reserved (context-dependent costing)
    /// Bit 4: Constant segregation flag
    /// Bit 3: Size flag
    /// Bits 2-0: Version (0-7)
    pub const HeaderType = packed struct(u8) {
        version: u3,
        has_size: bool,
        constant_segregation: bool,
        reserved1: bool = false,
        reserved_gzip: bool = false,
        multi_byte: bool = false,
    };
};
```

### SigmaBoolean

After evaluation, ErgoTree reduces to a **SigmaBoolean**—a tree of cryptographic propositions[^5][^6]:

```zig
const SigmaBoolean = union(enum) {
    prove_dlog: ProveDlog,           // Knowledge of discrete log
    prove_dh_tuple: ProveDhTuple,    // Diffie-Hellman tuple
    cand: Cand,                      // Logical AND
    cor: Cor,                        // Logical OR
    cthreshold: Cthreshold,          // k-of-n threshold
    trivial: TrivialProp,            // True/False

    /// Count nodes in proposition tree
    pub fn size(self: SigmaBoolean) usize {
        return switch (self) {
            .prove_dlog, .prove_dh_tuple, .trivial => 1,
            .cand => |c| 1 + totalChildrenSize(c.children),
            .cor => |c| 1 + totalChildrenSize(c.children),
            .cthreshold => |c| 1 + totalChildrenSize(c.children),
        };
    }
    // NOTE: In production, use an explicit work stack instead of recursion
    // to guarantee bounded stack depth. See ZIGMA_STYLE.md.
};

const ProveDlog = struct {
    /// Public key (compressed EC point, 33 bytes)
    h: EcPoint,
};

const ProveDhTuple = struct {
    g: EcPoint, // Generator
    h: EcPoint, // Point h
    u: EcPoint, // g^w
    v: EcPoint, // h^w
};
```

## The UTXO Model

Ergo uses the UTXO model where **boxes** are the fundamental units:

```
┌─────────────────────────────────────────┐
│                  Box                    │
├─────────────────────────────────────────┤
│  value: i64 (nanoERGs)                  │
│  ergoTree: ErgoTree                     │
│  tokens: []Token ──────────┐            │
│  registers: [10]?Constant  │            │
│  creationInfo: (height,    │            │
│                 txId, idx) │            │
└────────────────────────────┼────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│         Token           │   │   Register (R4-R9)      │
├─────────────────────────┤   ├─────────────────────────┤
│  id: [32]u8             │   │  value: Constant        │
│  amount: i64            │   │                         │
└─────────────────────────┘   └─────────────────────────┘
```

```zig
const Box = struct {
    value: i64,                          // nanoERGs (R0)
    ergo_tree: ErgoTree,                 // Spending condition (R1)
    tokens: []const Token,               // Additional assets (R2)
    creation_height: u32,                // Part of creation info (R3)
    tx_id: [32]u8,                       // Part of creation info (R3)
    output_index: u16,                   // Part of creation info (R3)
    additional_registers: [6]?Constant,  // R4-R9 (user-defined, optional)

    pub fn id(self: *const Box) [32]u8 {
        // Blake2b256(tx_id || output_index || serialized_content)
        var hasher = std.crypto.hash.blake2.Blake2b256.init(.{});
        hasher.update(&self.tx_id);
        hasher.update(std.mem.asBytes(&self.output_index));
        // ... serialize and hash content
        return hasher.finalResult();
    }
};
// NOTE: R0-R3 are computed from box fields; only R4-R9 are stored explicitly.
```

## The Prover/Verifier Model

```
                PROVER                              VERIFIER
          ┌──────────────┐                    ┌──────────────┐
          │   Secrets    │                    │              │
          │  (private    │                    │   Context    │
          │   keys)      │                    │              │
          └──────┬───────┘                    └──────┬───────┘
                 │                                   │
  ErgoTree ─────►│                    ErgoTree ─────►│
                 │                                   │
          ┌──────▼───────┐                    ┌──────▼───────┐
          │  Reduction   │                    │  Reduction   │
          │  (same as    │                    │  (same as    │
          │  verifier)   │                    │   prover)    │
          └──────┬───────┘                    └──────┬───────┘
                 │                                   │
          SigmaBoolean                        SigmaBoolean
                 │                                   │
          ┌──────▼───────┐                    ┌──────▼───────┐
          │   Signing    │───────Proof───────►│   Verify     │
          │ (Fiat-Shamir)│                    │  Signature   │
          └──────────────┘                    └──────┬───────┘
                                                     │
                                              true / false
```

### Prover

```zig
const Prover = struct {
    secrets: []const SecretKey,

    pub fn prove(
        self: *const Prover,
        ergo_tree: *const ErgoTree,
        context: *const Context,
    ) !Proof {
        // 1. Reduce to SigmaBoolean
        const sigma_bool = try Evaluator.reduce(ergo_tree, context);

        // 2. Generate proof using Fiat-Shamir
        return try self.generateProof(sigma_bool, context.message);
    }
};
```

### Verifier

```zig
const Verifier = struct {
    cost_limit: u64,

    pub fn verify(
        self: *const Verifier,
        ergo_tree: *const ErgoTree,
        context: *const Context,
        proof: *const Proof,
    ) !bool {
        // 1. Reduce with cost tracking
        var cost: u64 = 0;
        const sigma_bool = try Evaluator.reduceWithCost(
            ergo_tree, context, &cost, self.cost_limit,
        );

        // 2. Verify signature
        return try verifySignature(sigma_bool, proof, context.message);
    }
};
```

## Why Sigma Protocols?

Traditional scripts support only:
- Hash preimage revelation
- Signature verification
- Time locks

Sigma protocols enable:

| Feature | Description |
|---------|-------------|
| **Composable ZK Proofs** | AND, OR, threshold combinations |
| **Ring Signatures** | Prove set membership without revealing which |
| **Threshold Signatures** | k-of-n schemes |
| **Privacy** | Prove statements without full revelation |

```zig
// OR composition hides actual signer
const ring_signature = SigmaBoolean{
    .cor = .{
        .children = &[_]SigmaBoolean{
            .{ .prove_dlog = pk_alice },
            .{ .prove_dlog = pk_bob },
            .{ .prove_dlog = pk_carol },
        },
    },
};
// Proof reveals ONE signed, but not which
```

## Repository Structure

| Module | Purpose |
|--------|---------|
| `core` | Cryptographic primitives, base types |
| `data` | ErgoTree, AST nodes, serialization |
| `interpreter` | Evaluation engine, Sigma protocols |
| `parsers` | ErgoScript parser |
| `sc` | Compiler with IR optimization |
| `sdk` | High-level transaction APIs |

## Key Design Principles

### Determinism
All operations produce identical results for identical inputs—essential for consensus.

### Bounded Execution
Every script completes within a cost limit, tracking:
- Computational operations
- Memory allocations
- Cryptographic operations

### Soft-Fork Compatibility
ErgoTree includes version information; unknown opcodes are handled gracefully.

### Cross-Platform
Targets JVM (Scala), JavaScript (Scala.js), and Rust (sigma-rust)[^7].

## Summary

- **ErgoScript** compiles to **ErgoTree** bytecode
- **ErgoTree** evaluates to **SigmaBoolean** propositions
- **Prover** generates proofs; **Verifier** checks them
- **Sigma protocols** enable composable zero-knowledge proofs
- Design ensures **determinism**, **bounded execution**, **soft-fork compatibility**

---

*Next: [Chapter 2: Type System](./ch02-type-system.md)*

[^1]: Sigma protocols are interactive proof systems with the special "honest-verifier zero-knowledge" property.

[^2]: Scala: `docs/LangSpec.md:57-80`

[^3]: Scala: `data/shared/src/main/scala/sigma/ast/ErgoTree.scala:24-88`

[^4]: Rust: `ergotree-ir/src/ergo_tree/tree_header.rs:10-32`

[^5]: Scala: `core/shared/src/main/scala/sigma/data/SigmaBoolean.scala:12-21`

[^6]: Rust: `ergotree-ir/src/sigma_protocol/sigma_boolean.rs:34-80`

[^7]: Rust implementation: `sigma-rust` crate at `ergotree-ir/`, `ergotree-interpreter/`