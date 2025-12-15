# Chapter 23: Interpreter Wrappers

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Interpreter architecture ([Chapter 14](../part5/ch14-sigma-interpreter-architecture.md))
- Prover architecture ([Chapter 15](../part5/ch15-prover-architecture.md))
- Box model ([Chapter 22](../part7/ch22-box-model.md))

## Learning Objectives

- Understand the interpreter hierarchy
- Learn storage rent rules for expired boxes
- Master transaction signing with the Wallet API
- Implement proof verification

## Interpreter Architecture

The interpreter provides a layered architecture for script evaluation and proving[^1][^2]:

```
Interpreter Hierarchy
─────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────┐
│                    Verifier                         │
│  verify(tree, ctx, proof, message) -> bool          │
│  Evaluates tree, then verifies sigma protocol proof │
└────────────────────────┬────────────────────────────┘
                         │ uses
                         ▼
┌─────────────────────────────────────────────────────┐
│                    Prover                           │
│  prove(tree, ctx, message, hints) -> ProverResult   │
│  Reduces to SigmaBoolean, generates proof           │
├─────────────────────────────────────────────────────┤
│  secrets: []PrivateInput                            │
│  prove() generates commitment, response             │
└────────────────────────┬────────────────────────────┘
                         │ uses
                         ▼
┌─────────────────────────────────────────────────────┐
│               reduce_to_crypto                      │
│  Evaluates ErgoTree to SigmaBoolean                 │
│  Returns: { sigma_prop, cost, diag }                │
└─────────────────────────────────────────────────────┘
```

## Reduction to Crypto

The core evaluation function reduces ErgoTree to a cryptographic proposition[^3][^4]:

```zig
/// Result of expression reduction
const ReductionResult = struct {
    /// SigmaBoolean representing verifiable statement
    sigma_prop: SigmaBoolean,
    /// Estimated execution cost
    cost: u64,
    /// Diagnostic info (env state, pretty-printed expr)
    diag: ReductionDiagnosticInfo,
};

/// Evaluate ErgoTree to SigmaBoolean
pub fn reduceToCrypto(tree: *const ErgoTree, ctx: *const Context) !ReductionResult {
    const expr = try tree.root();
    var env = Env.empty();

    const value = try expr.eval(&env, ctx);

    const sigma_prop = switch (value) {
        .boolean => |b| SigmaBoolean.trivial(b),
        .sigma_prop => |sp| sp.value(),
        else => return error.NotSigmaProp,
    };

    return ReductionResult{
        .sigma_prop = sigma_prop,
        .cost = ctx.cost_accum.total(),
        .diag = .{
            .env = env.toStatic(),
            .pretty_printed_expr = null,
        },
    };
}
```

## Verifier Trait

Verification executes script and validates proof[^5][^6]:

```zig
const Verifier = struct {
    /// Verify proof against ErgoTree in context
    pub fn verify(
        self: *const Verifier,
        tree: *const ErgoTree,
        ctx: *const Context,
        proof: ProofBytes,
        message: []const u8,
    ) !VerificationResult {
        // Step 1-2: Reduce to SigmaBoolean
        const reduction = try reduceToCrypto(tree, ctx);

        // Step 3: Verify proof
        const result = switch (reduction.sigma_prop) {
            .trivial_prop => |b| b,
            else => |sb| blk: {
                if (proof.isEmpty()) break :blk false;

                // Parse signature and compute challenges
                const unchecked_tree = try parseSigComputeChallenges(
                    sb,
                    proof.bytes(),
                );

                // Verify commitments match
                break :blk try checkCommitments(unchecked_tree, message);
            },
        };

        return VerificationResult{
            .result = result,
            .cost = reduction.cost,
            .diag = reduction.diag,
        };
    }
};

const VerificationResult = struct {
    /// True if proof validates
    result: bool,
    /// Execution cost
    cost: u64,
    /// Diagnostic information
    diag: ReductionDiagnosticInfo,
};
```

## Prover Trait

The prover generates proofs for sigma propositions[^7][^8]:

```zig
const Prover = struct {
    /// Private inputs (secrets)
    secrets: []const PrivateInput,

    /// Generate proof for ErgoTree
    pub fn prove(
        self: *const Prover,
        tree: *const ErgoTree,
        ctx: *const Context,
        message: []const u8,
        hints: ?*const HintsBag,
    ) !ProverResult {
        // Reduce to crypto
        const reduction = try reduceToCrypto(tree, ctx);

        return switch (reduction.sigma_prop) {
            .trivial_prop => |b| if (b)
                ProverResult.empty()
            else
                error.ReducedToFalse,

            else => |sb| blk: {
                // Generate proof using sigma protocol
                const proof = try self.generateProof(sb, message, hints);
                break :blk proof;
            },
        };
    }

    /// Add secret to prover
    pub fn appendSecret(self: *Prover, secret: PrivateInput) void {
        self.secrets = append(self.secrets, secret);
    }

    /// Get public images of all secrets
    pub fn publicImages(self: *const Prover) []SigmaBoolean {
        var result: []SigmaBoolean = &.{};
        for (self.secrets) |secret| {
            result = append(result, secret.publicImage());
        }
        return result;
    }
};
```

## ProverResult

Proof output with context extension[^9][^10]:

```zig
const ProverResult = struct {
    /// Serialized proof bytes
    proof: ProofBytes,
    /// User-defined context variables
    extension: ContextExtension,

    pub fn empty() ProverResult {
        return .{
            .proof = ProofBytes.empty(),
            .extension = ContextExtension.empty(),
        };
    }
};

/// Proof bytes (empty for trivial proofs)
const ProofBytes = union(enum) {
    empty: void,
    some: []const u8,

    pub fn isEmpty(self: ProofBytes) bool {
        return self == .empty;
    }

    pub fn bytes(self: ProofBytes) []const u8 {
        return switch (self) {
            .empty => &.{},
            .some => |b| b,
        };
    }
};
```

## Wallet

The Wallet wraps prover for transaction signing[^11][^12]:

```zig
const Wallet = struct {
    /// Underlying prover
    prover: *Prover,

    /// Create from mnemonic phrase
    pub fn fromMnemonic(
        phrase: []const u8,
        password: []const u8,
    ) !Wallet {
        const seed = Mnemonic.toSeed(phrase, password);
        const ext_sk = try ExtSecretKey.deriveMaster(seed);
        return Wallet.fromSecrets(&.{ext_sk.secretKey()});
    }

    /// Create from secret keys
    pub fn fromSecrets(secrets: []const SecretKey) Wallet {
        var private_inputs: []PrivateInput = &.{};
        for (secrets) |sk| {
            private_inputs = append(private_inputs, PrivateInput.from(sk));
        }
        return .{
            .prover = &Prover{ .secrets = private_inputs },
        };
    }

    /// Add secret to wallet
    pub fn addSecret(self: *Wallet, secret: SecretKey) void {
        self.prover.appendSecret(PrivateInput.from(secret));
    }

    /// Sign a transaction
    pub fn signTransaction(
        self: *const Wallet,
        tx_context: *const TransactionContext,
        state_context: *const ErgoStateContext,
        tx_hints: ?*const TransactionHintsBag,
    ) !Transaction {
        return signTransactionImpl(
            self.prover,
            tx_context,
            state_context,
            tx_hints,
        );
    }

    /// Sign a reduced transaction
    pub fn signReducedTransaction(
        self: *const Wallet,
        reduced_tx: *const ReducedTransaction,
        tx_hints: ?*const TransactionHintsBag,
    ) !Transaction {
        return signReducedTransactionImpl(
            self.prover,
            reduced_tx,
            tx_hints,
        );
    }
};
```

## Transaction Signing

Sign all inputs, accumulating costs[^13][^14]:

```zig
/// Sign transaction, generating proofs for all inputs
pub fn signTransaction(
    prover: *const Prover,
    tx_context: *const TransactionContext,
    state_context: *const ErgoStateContext,
    tx_hints: ?*const TransactionHintsBag,
) !Transaction {
    const tx = tx_context.spending_tx;
    const message = try tx.bytesToSign();

    // Build context for first input
    var ctx = try makeContext(state_context, tx_context, 0);

    // Sign each input
    var inputs: []Input = &.{};
    for (tx.inputs(), 0..) |unsigned_input, idx| {
        if (idx > 0) {
            try updateContext(&ctx, tx_context, idx);
        }

        // Get hints for this input
        const hints = if (tx_hints) |h| h.allHintsForInput(idx) else null;

        // Generate proof
        const input_box = tx_context.getInputBox(unsigned_input.box_id) orelse
            return error.InputBoxNotFound;

        const prover_result = try prover.prove(
            &input_box.ergo_tree,
            &ctx,
            message,
            hints,
        );

        inputs = append(inputs, Input{
            .box_id = unsigned_input.box_id,
            .spending_proof = prover_result,
        });
    }

    return Transaction{
        .inputs = inputs,
        .data_inputs = tx.data_inputs,
        .output_candidates = tx.output_candidates,
    };
}

/// Create evaluation context for input
pub fn makeContext(
    state_ctx: *const ErgoStateContext,
    tx_ctx: *const TransactionContext,
    self_index: usize,
) !Context {
    const self_box = tx_ctx.getInputBox(
        tx_ctx.spending_tx.inputs()[self_index].box_id,
    ) orelse return error.InputBoxNotFound;

    return Context{
        .height = state_ctx.pre_header.height,
        .self_box = self_box,
        .outputs = tx_ctx.spending_tx.outputs(),
        .inputs = tx_ctx.inputBoxes(),
        .data_inputs = tx_ctx.dataBoxes(),
        .pre_header = state_ctx.pre_header,
        .headers = state_ctx.headers,
        .extension = tx_ctx.spending_tx.contextExtension(self_index),
    };
}
```

## Transaction Hints Bag

Hints for multi-party signing protocols[^15][^16]:

```zig
const TransactionHintsBag = struct {
    /// Secret hints by input index
    secret_hints: std.AutoHashMap(usize, HintsBag),
    /// Public hints (commitments) by input index
    public_hints: std.AutoHashMap(usize, HintsBag),

    pub fn empty() TransactionHintsBag {
        return .{
            .secret_hints = std.AutoHashMap(usize, HintsBag).init(allocator),
            .public_hints = std.AutoHashMap(usize, HintsBag).init(allocator),
        };
    }

    /// Replace all hints for an input
    pub fn replaceHintsForInput(self: *TransactionHintsBag, index: usize, hints: HintsBag) void {
        var public_hints: []Hint = &.{};
        var secret_hints: []Hint = &.{};

        for (hints.hints) |hint| {
            switch (hint) {
                .commitment_hint => public_hints = append(public_hints, hint),
                .secret_proven => secret_hints = append(secret_hints, hint),
            }
        }

        self.secret_hints.put(index, HintsBag{ .hints = secret_hints }) catch {};
        self.public_hints.put(index, HintsBag{ .hints = public_hints }) catch {};
    }

    /// Add hints for an input (appending to existing)
    pub fn addHintsForInput(self: *TransactionHintsBag, index: usize, hints: HintsBag) void {
        // Get existing or empty
        var existing_secret = self.secret_hints.get(index) orelse HintsBag.empty();
        var existing_public = self.public_hints.get(index) orelse HintsBag.empty();

        for (hints.hints) |hint| {
            switch (hint) {
                .commitment_hint => existing_public.hints = append(existing_public.hints, hint),
                .secret_proven => existing_secret.hints = append(existing_secret.hints, hint),
            }
        }

        self.secret_hints.put(index, existing_secret) catch {};
        self.public_hints.put(index, existing_public) catch {};
    }

    /// Get all hints for input
    pub fn allHintsForInput(self: *const TransactionHintsBag, index: usize) HintsBag {
        var hints: []Hint = &.{};

        if (self.secret_hints.get(index)) |bag| {
            for (bag.hints) |h| hints = append(hints, h);
        }
        if (self.public_hints.get(index)) |bag| {
            for (bag.hints) |h| hints = append(hints, h);
        }

        return HintsBag{ .hints = hints };
    }
};
```

## Commitment Generation

Generate first-round commitments for distributed signing[^17][^18]:

```zig
/// Generate commitments for transaction inputs
pub fn generateCommitments(
    wallet: *const Wallet,
    tx_context: *const TransactionContext,
    state_context: *const ErgoStateContext,
) !TransactionHintsBag {
    // Get public keys from wallet secrets
    var public_keys: []SigmaBoolean = &.{};
    for (wallet.prover.secrets) |secret| {
        public_keys = append(public_keys, secret.publicImage());
    }

    var hints_bag = TransactionHintsBag.empty();

    for (tx_context.spending_tx.inputs(), 0..) |_, idx| {
        var ctx = try makeContext(state_context, tx_context, idx);

        const input_box = tx_context.inputBoxes()[idx];
        const reduction = try reduceToCrypto(&input_box.ergo_tree, &ctx);

        // Generate commitments for this sigma proposition
        const input_hints = generateCommitmentsFor(
            &reduction.sigma_prop,
            public_keys,
        );
        hints_bag.addHintsForInput(idx, input_hints);
    }

    return hints_bag;
}
```

## Storage Rent (Ergo-Specific)

Boxes expire after ~4 years and can be spent by anyone[^19]:

```
Storage Rent Rules
─────────────────────────────────────────────────────

Period: 262,144 blocks ≈ 4 years (at 2 min/block)

Expired Box Spending:
┌─────────────────────────────────────────────────────┐
│ IF:                                                 │
│   current_height - box.creation_height >= 262,144  │
│   AND proof.isEmpty()                              │
│   AND extension.contains(STORAGE_INDEX_VAR)        │
│ THEN:                                              │
│   Check recreation rules instead of script         │
└─────────────────────────────────────────────────────┘

Recreation Rules:
┌─────────────────────────────────────────────────────┐
│ output.creation_height == current_height           │
│ output.value >= box.value - storage_fee            │
│ output.R1 == box.R1  (script preserved)            │
│ output.R2 == box.R2  (tokens preserved)            │
│ output.R4-R9 == box.R4-R9  (registers preserved)   │
│                                                    │
│ storage_fee = storage_fee_factor * box.bytes.len   │
└─────────────────────────────────────────────────────┘
```

```zig
const StorageConstants = struct {
    /// Storage period in blocks (~4 years)
    pub const STORAGE_PERIOD: u32 = 262_144;
    /// Context extension variable ID for storage index
    pub const STORAGE_INDEX_VAR_ID: u8 = 127;
    /// Fixed cost for storage contract evaluation
    pub const STORAGE_CONTRACT_COST: u64 = 50;
};

/// Check if expired box spending is valid
pub fn checkExpiredBox(
    box: *const ErgoBox,
    output: *const ErgoBoxCandidate,
    current_height: u32,
    storage_fee_factor: u64,
) bool {
    // Calculate storage fee
    const storage_fee = storage_fee_factor * box.serializedSize();

    // If box value <= fee, it's "dust" - always allowed
    if (box.value.as_i64() - @as(i64, @intCast(storage_fee)) <= 0) {
        return true;
    }

    // Check recreation rules
    const correct_height = output.creation_height == current_height;
    const correct_value = output.value.as_i64() >= box.value.as_i64() - @as(i64, @intCast(storage_fee));
    const correct_registers = checkRegistersPreserved(box, output);

    return correct_height and correct_value and correct_registers;
}

fn checkRegistersPreserved(box: *const ErgoBox, output: *const ErgoBoxCandidate) bool {
    // R0 (value) and R3 (reference) can change
    // R1 (script), R2 (tokens), R4-R9 must be preserved
    return eql(box.ergo_tree, output.ergo_tree) and
        eql(box.tokens, output.tokens) and
        eql(box.additional_registers, output.additional_registers);
}
```

## Signing Errors

```zig
const TxSigningError = error{
    /// Transaction context invalid
    TransactionContextError,
    /// Prover failed on input
    ProverError,
    /// Serialization failed
    SerializationError,
    /// Signature parsing failed
    SigParsingError,
};

const ProverError = error{
    /// ErgoTree parsing failed
    ErgoTreeError,
    /// Evaluation failed
    EvalError,
    /// Script reduced to false
    ReducedToFalse,
    /// Missing witness for proof
    TreeRootIsNotReal,
    /// Secret not found for leaf
    SecretNotFound,
    /// Simulated leaf needs challenge
    SimulatedLeafWithoutChallenge,
};
```

## Cost Tracking

Transaction costs are accumulated across inputs[^20]:

```zig
const TxCostComponents = struct {
    /// Interpreter initialization (once per tx)
    pub const INTERPRETER_INIT_COST: u64 = 10_000;

    /// Calculate total transaction cost
    pub fn calculateInitialCost(
        params: *const BlockchainParameters,
        inputs_count: usize,
        data_inputs_count: usize,
        outputs_count: usize,
        token_access_cost: u64,
    ) u64 {
        return INTERPRETER_INIT_COST +
            inputs_count * params.input_cost +
            data_inputs_count * params.data_input_cost +
            outputs_count * params.output_cost +
            token_access_cost;
    }
};
```

## Deterministic Signing

For platforms without secure random[^21][^22]:

```zig
/// Generate deterministic nonce from secret and message
/// Used when secure random is unavailable
pub fn generateDeterministicCommitments(
    wallet: *const Wallet,
    reduced_tx: *const ReducedTransaction,
    aux_rand: []const u8,
) !TransactionHintsBag {
    var hints_bag = TransactionHintsBag.empty();
    const message = try reduced_tx.unsigned_tx.bytesToSign();

    for (reduced_tx.reduced_inputs(), 0..) |input, idx| {
        // Deterministic nonce: H(secret || message || aux_rand)
        if (generateDeterministicCommitmentsFor(
            wallet.prover,
            &input.sigma_prop,
            message,
            aux_rand,
        )) |bag| {
            hints_bag.addHintsForInput(idx, bag);
        }
    }

    return hints_bag;
}
```

## Summary

- **Verifier** evaluates script, verifies sigma protocol proof
- **Prover** reduces to SigmaBoolean, generates proof using secrets
- **Wallet** wraps prover with transaction-level signing API
- **TransactionHintsBag** coordinates multi-party signing
- **Storage rent** allows expired boxes (~4 years) to be spent by anyone
- **Deterministic signing** available for platforms without secure random
- Cost accumulates across inputs with initial overhead

---

*Next: [Chapter 24: Transaction Validation](./ch24-transaction-validation.md)*

[^1]: Scala: `interpreter/shared/src/main/scala/org/ergoplatform/ErgoLikeInterpreter.scala`

[^2]: Rust: `ergotree-interpreter/src/eval.rs:1-50`

[^3]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala` (reduce)

[^4]: Rust: `ergotree-interpreter/src/eval.rs:129-160`

[^5]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/Interpreter.scala` (verify)

[^6]: Rust: `ergotree-interpreter/src/sigma_protocol/verifier.rs:55-88`

[^7]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/ProverInterpreter.scala`

[^8]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs:57-96`

[^9]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/ProverResult.scala`

[^10]: Rust: `ergo-lib/src/chain/transaction/input/prover_result.rs:14-50`

[^11]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoProvingInterpreter.scala`

[^12]: Rust: `ergo-lib/src/wallet.rs:52-94`

[^13]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoProvingInterpreter.scala:100-159`

[^14]: Rust: `ergo-lib/src/wallet/signing.rs:143-180`

[^15]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/HintsBag.scala`

[^16]: Rust: `ergo-lib/src/wallet.rs:259-347`

[^17]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoProvingInterpreter.scala:190-217`

[^18]: Rust: `ergo-lib/src/wallet.rs:124-158`

[^19]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoInterpreter.scala:42-55`

[^20]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoInterpreter.scala:93-96`

[^21]: Rust: `ergo-lib/src/wallet.rs:182-209`

[^22]: Rust: `ergo-lib/src/wallet/deterministic.rs`