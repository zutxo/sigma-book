# Chapter 14: Verifier Implementation

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 11](../part4/ch11-sigma-protocols.md) for Sigma protocol verification and Fiat-Shamir transformation
- [Chapter 12](./ch12-evaluation-model.md) for ErgoTree reduction to SigmaBoolean
- [Chapter 13](./ch13-cost-model.md) for cost accumulation during verification

## Learning Objectives

By the end of this chapter, you will be able to:

- Trace the complete verification flow from ErgoTree to boolean result
- Implement `verify()` and `fullReduction()` methods
- Handle soft-fork conditions gracefully to maintain network compatibility
- Verify cryptographic signatures using Fiat-Shamir commitment reconstruction
- Estimate verification cost before performing expensive cryptographic operations

## Verification Overview

Verification is the counterpart to proving: given an ErgoTree, a transaction context, and a cryptographic proof, the verifier determines whether the proof is valid. This process happens for every input box in every transaction—efficient verification is critical for blockchain throughput.

The verification proceeds in two phases: first reduce the ErgoTree to a SigmaBoolean proposition (using the evaluator from Chapter 12), then verify the cryptographic proof satisfies that proposition[^1][^2].

```
Verification Pipeline
─────────────────────────────────────────────────────

Input: ErgoTree + Context + Proof + Message

┌──────────────────────────────────────────────────┐
│              1. REDUCTION PHASE                  │
│                                                  │
│  ErgoTree ────> propositionFromErgoTree()        │
│                        │                         │
│                        ▼                         │
│              SigmaPropValue                      │
│                        │                         │
│                        ▼                         │
│              fullReduction()                     │
│                        │                         │
│                        ▼                         │
│              SigmaBoolean + Cost                 │
└──────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│            2. VERIFICATION PHASE                 │
│                                                  │
│  TrueProp  ────> return (true, cost)             │
│  FalseProp ────> return (false, cost)            │
│                                                  │
│  Otherwise:                                      │
│    estimateCryptoVerifyCost()                    │
│              │                                   │
│              ▼                                   │
│    verifySignature() ────> boolean result        │
└──────────────────────────────────────────────────┘

Output: (verified: bool, total_cost: u64)
```

## Verification Result

```zig
const VerificationResult = struct {
    /// Result of SigmaProp verification
    result: bool,
    /// Estimated cost of contract execution
    cost: u64,
    /// Diagnostic information
    diag: ReductionDiagnosticInfo,
};

const ReductionResult = struct {
    /// SigmaBoolean proposition
    sigma_prop: SigmaBoolean,
    /// Accumulated cost (block scale)
    cost: u64,
    /// Diagnostic info
    diag: ReductionDiagnosticInfo,
};

const ReductionDiagnosticInfo = struct {
    /// Environment after evaluation
    env: Env,
    /// Pretty-printed expression
    pretty_printed_expr: ?[]const u8,
};
```

## Verifier Trait

The base verifier interface[^3][^4]:

```zig
const Verifier = struct {
    const Self = @This();

    /// Cost per byte for deserialization
    pub const COST_PER_BYTE_DESERIALIZED: i32 = 2;
    /// Cost per tree byte for substitution
    pub const COST_PER_TREE_BYTE: i32 = 2;

    /// Verify an ErgoTree in context with proof
    pub fn verify(
        self: *const Self,
        ergo_tree: *const ErgoTree,
        context: *const Context,
        proof: ProofBytes,
        message: []const u8,
    ) VerifierError!VerificationResult {
        // Reduce to SigmaBoolean
        const reduction = try reduceToCrypto(ergo_tree, context);

        const result: bool = switch (reduction.sigma_prop) {
            .trivial_prop => |b| b,
            else => |sb| blk: {
                if (proof.isEmpty()) {
                    break :blk false;
                }
                // Verifier Steps 1-3: Parse proof
                const unchecked = try parseAndComputeChallenges(&sb, proof.bytes());
                // Verifier Steps 4-6: Check commitments
                break :blk try checkCommitments(unchecked, message);
            },
        };

        return .{
            .result = result,
            .cost = reduction.cost,
            .diag = reduction.diag,
        };
    }
};
```

## The verify() Method

Complete verification entry point[^5]:

```zig
pub fn verify(
    env: ScriptEnv,
    ergo_tree: *const ErgoTree,
    context: *const Context,
    proof: []const u8,
    message: []const u8,
) VerifierError!VerificationResult {
    // Check soft-fork condition first
    if (checkSoftForkCondition(ergo_tree, context)) |soft_fork_result| {
        return soft_fork_result;
    }

    // REDUCTION PHASE
    const reduced = try fullReduction(ergo_tree, context, env);

    // VERIFICATION PHASE
    return switch (reduced.sigma_prop) {
        .true_prop => .{ .result = true, .cost = reduced.cost, .diag = reduced.diag },
        .false_prop => .{ .result = false, .cost = reduced.cost, .diag = reduced.diag },
        else => |sb| blk: {
            // Non-trivial proposition: verify cryptographic proof
            const full_cost = try addCryptoCost(sb, reduced.cost, context.cost_limit);

            const ok = verifySignature(sb, message, proof) catch false;
            break :blk .{
                .result = ok,
                .cost = full_cost,
                .diag = reduced.diag,
            };
        },
    };
}
```

## Full Reduction

Reduces ErgoTree to SigmaBoolean with cost tracking[^6][^7]:

```zig
pub fn fullReduction(
    ergo_tree: *const ErgoTree,
    context: *const Context,
    env: ScriptEnv,
) ReducerError!ReductionResult {
    // Extract proposition from ErgoTree
    const prop = try propositionFromErgoTree(ergo_tree, context);

    // Fast path: SigmaProp constant
    if (prop == .sigma_prop_constant) {
        const sb = prop.sigma_prop_constant.toSigmaBoolean();
        const eval_cost = SigmaPropConstant.COST.cost.toBlockCost();
        const res_cost = try addCostChecked(context.init_cost, eval_cost, context.cost_limit);
        return .{
            .sigma_prop = sb,
            .cost = res_cost,
            .diag = .{ .env = context.env, .pretty_printed_expr = null },
        };
    }

    // No DeserializeContext: direct evaluation
    if (!ergo_tree.hasDeserialize()) {
        return evalToCrypto(context, ergo_tree);
    }

    // Has DeserializeContext: special handling
    return reductionWithDeserialize(ergo_tree, prop, context, env);
}

fn propositionFromErgoTree(
    ergo_tree: *const ErgoTree,
    context: *const Context,
) PropositionError!SigmaPropValue {
    return switch (ergo_tree.root) {
        .parsed => |tree| ergo_tree.toProposition(ergo_tree.header.constant_segregation),
        .unparsed => |u| blk: {
            if (context.validation_settings.isSoftFork(u.err)) {
                // Soft-fork: return true (accept)
                break :blk SigmaPropValue.true_sigma_prop;
            }
            // Hard error
            return error.UnparsedErgoTree;
        },
    };
}
```

## Signature Verification

Implements Verifier Steps 4-6 of the Sigma protocol[^8][^9]:

```zig
/// Verify a signature on message for given proposition
pub fn verifySignature(
    sigma_tree: SigmaBoolean,
    message: []const u8,
    signature: []const u8,
) VerifierError!bool {
    return switch (sigma_tree) {
        .trivial_prop => |b| b,
        else => |sb| blk: {
            if (signature.len == 0) {
                break :blk false;
            }
            // Verifier Steps 1-3: Parse proof
            const unchecked = try parseAndComputeChallenges(&sb, signature);
            // Verifier Steps 4-6: Check commitments
            break :blk try checkCommitments(unchecked, message);
        },
    };
}

/// Verifier Steps 4-6: Check commitments match Fiat-Shamir challenge
fn checkCommitments(
    sp: UncheckedTree,
    message: []const u8,
) VerifierError!bool {
    // Verifier Step 4: Compute commitments from challenges and responses
    const new_root = computeCommitments(sp);

    // Steps 5-6: Serialize tree for Fiat-Shamir
    var buf = std.ArrayList(u8).init(allocator);
    try fiatShamirTreeToBytes(&new_root, buf.writer());
    try buf.appendSlice(message);

    // Compute expected challenge
    const expected_challenge = fiatShamirHashFn(buf.items);

    // Compare with actual challenge
    // NOTE: In production, use constant-time comparison for challenge bytes
    // to prevent timing side-channels: std.crypto.utils.timingSafeEql
    return std.mem.eql(u8, &new_root.challenge(), &expected_challenge);
}
```

## Computing Commitments

Verifier Step 4: Reconstruct commitments from challenges and responses[^10][^11]:

```zig
/// For every leaf, compute commitment from challenge and response
pub fn computeCommitments(sp: UncheckedTree) UncheckedTree {
    return switch (sp) {
        .unchecked_leaf => |leaf| switch (leaf) {
            .unchecked_schnorr => |sn| blk: {
                // Reconstruct: a = g^z / h^e
                const a = DlogProver.computeCommitment(
                    &sn.proposition,
                    &sn.challenge,
                    &sn.second_message,
                );
                break :blk UncheckedTree{
                    .unchecked_leaf = .{
                        .unchecked_schnorr = .{
                            .proposition = sn.proposition,
                            .challenge = sn.challenge,
                            .second_message = sn.second_message,
                            .commitment_opt = FirstDlogProverMessage{ .a = a },
                        },
                    },
                };
            },
            .unchecked_dh_tuple => |dh| blk: {
                // Reconstruct both commitments
                const commitment = DhTupleProver.computeCommitment(
                    &dh.proposition,
                    &dh.challenge,
                    &dh.second_message,
                );
                break :blk UncheckedTree{
                    .unchecked_leaf = .{
                        .unchecked_dh_tuple = .{
                            .proposition = dh.proposition,
                            .challenge = dh.challenge,
                            .second_message = dh.second_message,
                            .commitment_opt = commitment,
                        },
                    },
                };
            },
        },
        .unchecked_conjecture => |conj| blk: {
            // Recursively process children
            var new_children = allocator.alloc(UncheckedTree, conj.children.len);
            for (conj.children, 0..) |child, i| {
                new_children[i] = computeCommitments(child);
            }
            break :blk conj.withChildren(new_children);
        },
    };
}
```

## Crypto Verification Cost

Estimate cost before performing expensive operations[^12]:

```zig
const VerificationCosts = struct {
    /// Cost for Schnorr commitment computation
    pub const COMPUTE_COMMITMENTS_SCHNORR = FixedCost{ .cost = .{ .value = 3400 } };
    /// Cost for DHT commitment computation
    pub const COMPUTE_COMMITMENTS_DHT = FixedCost{ .cost = .{ .value = 6450 } };

    /// Total Schnorr verification cost
    pub const PROVE_DLOG_VERIFICATION: JitCost = blk: {
        const parse = ParseChallenge_ProveDlog.COST.cost;
        const compute = COMPUTE_COMMITMENTS_SCHNORR.cost;
        const serialize = ToBytes_Schnorr.COST.cost;
        break :blk parse.add(compute).add(serialize);
    };

    /// Total DHT verification cost
    pub const PROVE_DHT_VERIFICATION: JitCost = blk: {
        const parse = ParseChallenge_ProveDHT.COST.cost;
        const compute = COMPUTE_COMMITMENTS_DHT.cost;
        const serialize = ToBytes_DHT.COST.cost;
        break :blk parse.add(compute).add(serialize);
    };
};

/// Estimate verification cost without performing crypto
pub fn estimateCryptoVerifyCost(sb: SigmaBoolean) JitCost {
    return switch (sb) {
        .prove_dlog => VerificationCosts.PROVE_DLOG_VERIFICATION,
        .prove_dh_tuple => VerificationCosts.PROVE_DHT_VERIFICATION,
        .c_and => |and_node| blk: {
            const node_cost = ToBytes_ProofTreeConjecture.COST.cost;
            var children_cost = JitCost{ .value = 0 };
            for (and_node.children) |child| {
                children_cost = children_cost.add(estimateCryptoVerifyCost(child)) catch unreachable;
            }
            break :blk node_cost.add(children_cost) catch unreachable;
        },
        .c_or => |or_node| blk: {
            const node_cost = ToBytes_ProofTreeConjecture.COST.cost;
            var children_cost = JitCost{ .value = 0 };
            for (or_node.children) |child| {
                children_cost = children_cost.add(estimateCryptoVerifyCost(child)) catch unreachable;
            }
            break :blk node_cost.add(children_cost) catch unreachable;
        },
        .c_threshold => |th| blk: {
            const n_children = th.children.len;
            const n_coefs = n_children - th.k;
            const parse_cost = ParsePolynomial.COST.cost(@intCast(n_coefs));
            const eval_cost = EvaluatePolynomial.COST.cost(@intCast(n_coefs)).mul(@intCast(n_children)) catch unreachable;
            const node_cost = ToBytes_ProofTreeConjecture.COST.cost;
            var children_cost = JitCost{ .value = 0 };
            for (th.children) |child| {
                children_cost = children_cost.add(estimateCryptoVerifyCost(child)) catch unreachable;
            }
            break :blk parse_cost.add(eval_cost).add(node_cost).add(children_cost) catch unreachable;
        },
        else => JitCost{ .value = 0 }, // Trivial proposition
    };
}

/// Add crypto cost to accumulated cost
fn addCryptoCost(
    sigma_prop: SigmaBoolean,
    base_cost: u64,
    cost_limit: u64,
) CostError!u64 {
    const crypto_cost = estimateCryptoVerifyCost(sigma_prop).toBlockCost();
    return addCostChecked(base_cost, crypto_cost, cost_limit);
}
```

## Soft-Fork Handling

Handle unrecognized script versions gracefully[^13]:

```zig
/// Check for soft-fork condition
fn checkSoftForkCondition(
    ergo_tree: *const ErgoTree,
    context: *const Context,
) ?VerificationResult {
    if (context.activated_script_version > MAX_SUPPORTED_SCRIPT_VERSION) {
        // Protocol version exceeds interpreter capabilities
        if (ergo_tree.header.version > MAX_SUPPORTED_SCRIPT_VERSION) {
            // Cannot verify: accept and rely on 90% upgraded nodes
            return .{
                .result = true,
                .cost = context.init_cost,
                .diag = .{ .env = Env.empty(), .pretty_printed_expr = null },
            };
        }
        // Can verify despite protocol upgrade
    } else {
        // Activated version within supported range
        if (ergo_tree.header.version > context.activated_script_version) {
            // ErgoTree version too high
            return error.ErgoTreeVersionTooHigh;
        }
    }
    return null; // Proceed normally
}

/// Soft-fork reduction result: accept as true
fn whenSoftForkReductionResult(cost: u64) ReductionResult {
    return .{
        .sigma_prop = .{ .trivial_prop = true },
        .cost = cost,
        .diag = .{ .env = Env.empty(), .pretty_printed_expr = null },
    };
}
```

## DeserializeContext Handling

Scripts may contain deserialization operations[^14]:

```zig
fn reductionWithDeserialize(
    ergo_tree: *const ErgoTree,
    prop: SigmaPropValue,
    context: *const Context,
    env: ScriptEnv,
) ReducerError!ReductionResult {
    // Add cost for deserialization substitution
    const tree_bytes = ergo_tree.bytes();
    const deserialize_cost = @as(i64, @intCast(tree_bytes.len)) * COST_PER_TREE_BYTE;
    const curr_cost = try addCostChecked(context.init_cost, deserialize_cost, context.cost_limit);

    var context1 = context.*;
    context1.init_cost = curr_cost;

    // Substitute DeserializeContext nodes
    const prop_tree = try applyDeserializeContext(&context1, prop);

    // Reduce the substituted tree
    return reduceToCrypto(&context1, prop_tree);
}
```

## Complete Verification Flow

```
verify(ergoTree, context, proof, message)
─────────────────────────────────────────────────────

Step 1: checkSoftForkCondition()
        │
        ├─ activated > MaxSupported AND script > MaxSupported
        │  └─> return (true, initCost)  [soft-fork accept]
        │
        ├─ script.version > activated
        │  └─> throw ErgoTreeVersionTooHigh
        │
        └─ Otherwise: proceed
                │
                ▼
Step 2: fullReduction()
        │
        ├─ propositionFromErgoTree()
        │  └─ Handle unparsed trees
        │
        ├─ SigmaPropConstant
        │  └─> Extract directly
        │
        ├─ No DeserializeContext
        │  └─> evalToCrypto()
        │
        └─ Has DeserializeContext
           └─> reductionWithDeserialize()
                │
                ▼
        ReductionResult(sigmaBoolean, cost)
                │
                ▼
Step 3: Check result
        │
        ├─ TrueProp  ────> return (true, cost)
        ├─ FalseProp ────> return (false, cost)
        └─ Non-trivial ────> continue
                │
                ▼
Step 4: addCryptoCost()
        │
        └─ Estimate without crypto ops
                │
                ▼
Step 5: verifySignature()
        │
        ├─ parseAndComputeChallenges()
        │  └─ Parse proof bytes
        │
        ├─ computeCommitments()
        │  └─ Reconstruct commitments
        │
        ├─ fiatShamirTreeToBytes()
        │  └─ Serialize tree
        │
        └─ fiatShamirHashFn()
           └─ Compute expected challenge
                │
                ▼
Step 6: Return (result, totalCost)
```

## Verifier Errors

```zig
const VerifierError = error{
    /// Failed to parse ErgoTree
    ErgoTreeError,
    /// Failed to evaluate ErgoTree
    EvalError,
    /// Signature parsing error
    SigParsingError,
    /// Fiat-Shamir serialization error
    FiatShamirTreeSerializationError,
    /// Cost limit exceeded
    CostLimitExceeded,
    /// ErgoTree version too high
    ErgoTreeVersionTooHigh,
    /// Cannot parse unparsed tree
    UnparsedErgoTree,
};
```

## Test Verifier

Simple verifier implementation for testing[^15]:

```zig
const TestVerifier = struct {
    const Self = @This();

    pub fn verify(
        self: *const Self,
        tree: *const ErgoTree,
        ctx: *const Context,
        proof: ProofBytes,
        message: []const u8,
    ) VerifierError!VerificationResult {
        _ = self;
        const reduction = try reduceToCrypto(tree, ctx);

        const result: bool = switch (reduction.sigma_prop) {
            .trivial_prop => |b| b,
            else => |sb| blk: {
                if (proof.isEmpty()) {
                    break :blk false;
                }
                const unchecked = try parseAndComputeChallenges(&sb, proof.bytes());
                break :blk try checkCommitments(unchecked, message);
            },
        };

        return .{
            .result = result,
            .cost = 0, // Test verifier doesn't track cost
            .diag = reduction.diag,
        };
    }
};
```

## Summary

This chapter covered the verifier implementation that validates Sigma proofs:

- **Verification proceeds in two phases**: reduction (ErgoTree → SigmaBoolean) and cryptographic verification (proof checking)
- **`fullReduction()`** evaluates the ErgoTree to a SigmaBoolean proposition while tracking costs
- **`verifySignature()`** implements Verifier Steps 4-6: parse proof bytes, compute expected commitments from challenges and responses, then verify via Fiat-Shamir hash
- **Soft-fork handling** accepts scripts with unrecognized versions or opcodes, enabling protocol upgrades without network splits
- **Cost estimation** predicts cryptographic verification cost before performing expensive EC operations, failing early if the limit would be exceeded
- **Commitment reconstruction** (`computeCommitments`) derives the prover's commitments from the challenges and responses, which must match the Fiat-Shamir challenge
- **`DeserializeContext`** nodes are substituted with their deserialized values before reduction begins

---

*Next: [Chapter 15: Prover Implementation](./ch15-prover-implementation.md)*

[^1]: Scala: [`Interpreter.scala:30-100`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L30-L100)

[^2]: Rust: [`verifier.rs:27-52`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/verifier.rs#L27-L52)

[^3]: Scala: [`Interpreter.scala:78-92`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L78-L92)

[^4]: Rust: [`verifier.rs:55-88`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/verifier.rs#L55-L88)

[^5]: Scala: [`Interpreter.scala:132-167`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L132-L167)

[^6]: Scala: [`Interpreter.scala:196-239`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L196-L239)

[^7]: Rust: [`eval.rs:130-160`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval.rs#L130-L160)

[^8]: Scala: [`Interpreter.scala:282-298`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L282-L298)

[^9]: Rust: [`verifier.rs:91-125`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/verifier.rs#L91-L125)

[^10]: Scala: [`Interpreter.scala:324-347`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L324-L347)

[^11]: Rust: [`verifier.rs:127-163`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/verifier.rs#L127-L163)

[^12]: Scala: [`Interpreter.scala:362-408`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L362-L408)

[^13]: Scala: [`Interpreter.scala:450-472`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L450-L472)

[^14]: Scala: [`Interpreter.scala:492-517`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala#L492-L517)

[^15]: Rust: [`verifier.rs:166-168`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/sigma_protocol/verifier.rs#L166-L168)