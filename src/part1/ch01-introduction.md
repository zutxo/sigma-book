# Chapter 1: Introduction to Sigma and ErgoTree

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Basic blockchain concepts (transactions, blocks, consensus)
- Understanding of the UTXO model (unspent transaction outputs)
- Familiarity with any systems programming language (C, Rust, Go, or similar)
- Public key cryptography fundamentals (key pairs, digital signatures, hash functions)

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why Sigma protocols offer advantages over traditional blockchain scripting
- Describe the relationship between ErgoScript, ErgoTree, and SigmaBoolean
- Understand the UTXO model and how scripts guard spending conditions
- Differentiate the roles of prover and verifier in transaction validation
- Identify the core components of the Sigma interpreter architecture

## What is Sigma?

Traditional blockchain scripting languages like Bitcoin Script offer limited expressiveness: they support hash preimages, signature checks, and timelocks, but little else. Ethereum's EVM provides Turing completeness but at the cost of complexity, high gas fees, and limited privacy guarantees.

**Sigma** (Σ) protocols occupy a middle ground. They are cryptographic proof systems that enable zero-knowledge proofs of knowledge—proving you know a secret without revealing it[^1]. The name comes from the Greek letter Σ and reflects their characteristic three-move structure:

1. **Commitment**: The prover sends a randomized commitment value
2. **Challenge**: The verifier sends a random challenge
3. **Response**: The prover sends a response that proves knowledge without revealing secrets

What makes Sigma protocols powerful for blockchains is their **composability**: you can combine them with AND, OR, and threshold operations to build complex spending conditions while preserving zero-knowledge properties.

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

Ergo extends the UTXO (Unspent Transaction Output) model pioneered by Bitcoin. Instead of simple locking scripts, Ergo uses **boxes**—rich data structures that contain value, tokens, and arbitrary typed data:

```
┌─────────────────────────────────────────┐
│                  Box                    │
├─────────────────────────────────────────┤
│  R0: value (i64 nanoERGs)               │  ← Computed registers
│  R1: ergoTree (spending condition)      │
│  R2: tokens (asset list)                │
│  R3: creationInfo (height, txId, idx)   │
├─────────────────────────────────────────┤
│  R4-R9: additional registers ───────────┼──► User-defined data
│         (optional, typed constants)     │     (up to 6 registers)
└─────────────────────────────────────────┘
                    │
     ┌──────────────┴──────────────┐
     ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│         Token           │   │   Register (R4-R9)      │
├─────────────────────────┤   ├─────────────────────────┤
│  id: [32]u8 (token ID)  │   │  value: Constant        │
│  amount: i64            │   │  (any SType value)      │
└─────────────────────────┘   └─────────────────────────┘
```

Registers R0–R3 are computed from box fields and always present. Registers R4–R9 are optional and can store any typed value—integers, byte arrays, group elements, or even nested collections.

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
TODO: Add explainations.

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
TODO: Add explainations.
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
TODO: Add explainations.
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

Consider what Bitcoin Script can express: "This output can be spent if you provide a valid signature for public key X." This covers most payment scenarios but falls short for more sophisticated applications.

Sigma protocols enable a fundamentally richer set of spending conditions:

| Feature | What It Enables | Example Use Case |
|---------|-----------------|------------------|
| **Composable ZK Proofs** | AND, OR, threshold combinations of conditions | Multi-party escrow with complex release logic |
| **Ring Signatures** | Prove you're one of N signers without revealing which | Anonymous voting, whistleblower systems |
| **Threshold Signatures** | Require k-of-n parties to sign | DAO governance, cold storage recovery |
| **Zero-Knowledge Privacy** | Prove statements without revealing underlying data | Private auctions, confidential identity verification |

The key insight is that Sigma protocols can be **composed** while preserving their zero-knowledge properties. An OR composition of two Sigma proofs reveals that the prover knows one of two secrets—but not which one.

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

The Sigma interpreter is built around four core principles that make it suitable for blockchain consensus:

### Determinism
Every operation must produce identical results for identical inputs, regardless of platform or implementation. This means no floating-point arithmetic, no uninitialized memory, and careful handling of hash map iteration order. Without determinism, nodes would disagree on transaction validity.

### Bounded Execution
Every script must complete within a predictable cost limit. The interpreter tracks three resource categories:
- **Computational operations**: arithmetic, comparisons, function calls
- **Memory allocations**: collections, tuples, intermediate values
- **Cryptographic operations**: EC point multiplication, signature verification

Scripts exceeding the cost limit fail validation, preventing denial-of-service attacks.

### Soft-Fork Compatibility
ErgoTree includes version information in its header. When nodes encounter unknown opcodes (from future protocol versions), they can handle them gracefully rather than rejecting the entire block. This enables protocol upgrades without hard forks.

### Cross-Platform Consistency
The specification must be implementable identically across different platforms. Reference implementations exist for:
- **JVM** (Scala): The original sigmastate-interpreter
- **JavaScript** (Scala.js): Browser and Node.js environments
- **Native** (Rust): sigma-rust for performance-critical applications[^7]

## Summary

This chapter introduced the fundamental concepts of the Sigma protocol ecosystem:

- **Sigma protocols** are three-move cryptographic proofs that enable zero-knowledge proofs of knowledge, with the crucial property of composability
- **ErgoScript** is a high-level, statically-typed language that compiles to **ErgoTree** bytecode
- **ErgoTree** is a serialized AST stored in UTXO boxes that evaluates to **SigmaBoolean** propositions
- **SigmaBoolean** represents cryptographic conditions (discrete log proofs, Diffie-Hellman tuples) combined with AND, OR, and threshold logic
- The **prover** generates zero-knowledge proofs; the **verifier** checks them without learning secrets
- The system is designed for blockchain consensus: **deterministic**, **bounded**, **soft-fork compatible**, and **cross-platform**

In the following chapters, we'll dive deep into each layer—starting with the type system that makes ErgoTree's static guarantees possible.

---

*Next: [Chapter 2: Type System](./ch02-type-system.md)*

[^1]: Sigma protocols are interactive proof systems with the special "honest-verifier zero-knowledge" property.

[^2]: Scala: [`LangSpec.md:57-80`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/docs/LangSpec.md#L57-L80)

[^3]: Scala: [`ErgoTree.scala:24-88`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/ErgoTree.scala#L24-L88)

[^4]: Rust: [`tree_header.rs:10-32`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/ergo_tree/tree_header.rs#L10-L32)

[^5]: Scala: [`SigmaBoolean.scala:12-21`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/SigmaBoolean.scala#L12-L21)

[^6]: Rust: [`sigma_boolean.rs:34-80`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/sigma_protocol/sigma_boolean.rs#L34-L80)

[^7]: Rust implementation: [sigma-rust](https://github.com/ergoplatform/sigma-rust) crate at `ergotree-ir/`, `ergotree-interpreter/`
