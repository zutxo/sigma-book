# Chapter 26: Wallet and Signing

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 15](../part5/ch15-prover-implementation.md) for proof generation
- [Chapter 23](./ch23-interpreter-wrappers.md) for interpreter integration
- [Chapter 11](../part4/ch11-sigma-protocols.md) for hint system and distributed signing

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the wallet service architecture and its role in transaction signing
- Trace the complete transaction signing flow from unsigned to signed
- Use `TransactionHintsBag` for distributed multi-party signing
- Implement box selection strategies for building transactions

## Wallet Architecture

The wallet bridges high-level operations with the interpreter layer[^1][^2]:

```
Wallet Service Architecture
─────────────────────────────────────────────────────────

┌────────────────────────────────────────────────────────┐
│                   Wallet                               │
├────────────────────────────────────────────────────────┤
│  prover: Box<dyn Prover>                               │
│                                                        │
│  ├── from_mnemonic(phrase, pass) -> Wallet             │
│  ├── from_secrets([]SecretKey) -> Wallet               │
│  ├── add_secret(SecretKey)                             │
│  │                                                     │
│  ├── sign_transaction(...) -> Transaction              │
│  ├── sign_reduced_transaction(...) -> Transaction      │
│  │                                                     │
│  └── generate_commitments(...) -> TransactionHintsBag  │
└────────────────────────────────────────────────────────┘
                        │
                        │ uses
                        ▼
┌────────────────────────────────────────────────────────┐
│                    Prover                              │
│  prove(tree, ctx, message, hints) -> ProverResult      │
└────────────────────────────────────────────────────────┘
```

## Wallet Structure

```zig
const Wallet = struct {
    /// Underlying prover (boxed trait object)
    prover: *Prover,
    allocator: Allocator,

    /// Create wallet from mnemonic phrase
    pub fn fromMnemonic(
        mnemonic_phrase: []const u8,
        mnemonic_pass: []const u8,
        allocator: Allocator,
    ) !Wallet {
        const seed = Mnemonic.toSeed(mnemonic_phrase, mnemonic_pass);
        const ext_sk = try ExtSecretKey.deriveMaster(seed);
        return Wallet.fromSecrets(&.{ext_sk.secretKey()}, allocator);
    }

    /// Create wallet from secret keys
    pub fn fromSecrets(secrets: []const SecretKey, allocator: Allocator) Wallet {
        var private_inputs = allocator.alloc(PrivateInput, secrets.len) catch unreachable;
        for (secrets, 0..) |sk, i| {
            private_inputs[i] = PrivateInput.from(sk);
        }
        return .{
            .prover = TestProver.init(private_inputs, allocator),
            .allocator = allocator,
        };
    }

    /// Add secret to wallet
    pub fn addSecret(self: *Wallet, secret: SecretKey) void {
        self.prover.appendSecret(PrivateInput.from(secret));
    }

    /// Sign a transaction
    pub fn signTransaction(
        self: *const Wallet,
        tx_context: *const TransactionContext(UnsignedTransaction),
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

    /// Generate commitments for distributed signing
    pub fn generateCommitments(
        self: *const Wallet,
        tx_context: *const TransactionContext(UnsignedTransaction),
        state_context: *const ErgoStateContext,
    ) !TransactionHintsBag {
        var public_keys: []SigmaBoolean = &.{};
        for (self.prover.secrets()) |secret| {
            public_keys = append(public_keys, secret.publicImage());
        }
        return generateCommitmentsImpl(tx_context, state_context, public_keys);
    }
};
```

## Mnemonic Seed Generation

BIP-39 mnemonic to seed conversion[^3][^4]:

```zig
const Mnemonic = struct {
    /// PBKDF2 iterations per BIP-39
    pub const PBKDF2_ITERATIONS: u32 = 2048;
    /// Seed output length (SHA-512)
    pub const SEED_LENGTH: usize = 64;

    /// Convert mnemonic phrase to seed bytes
    pub fn toSeed(
        mnemonic_phrase: []const u8,
        mnemonic_pass: []const u8,
    ) [SEED_LENGTH]u8 {
        var seed: [SEED_LENGTH]u8 = undefined;

        // Normalize to NFKD form
        const normalized_phrase = normalizeNfkd(mnemonic_phrase);
        const normalized_pass = normalizeNfkd(mnemonic_pass);

        // Salt is "mnemonic" + password
        var salt_buf: [256]u8 = undefined;
        const salt_prefix = "mnemonic";
        @memcpy(salt_buf[0..salt_prefix.len], salt_prefix);
        @memcpy(salt_buf[salt_prefix.len..][0..normalized_pass.len], normalized_pass);
        const salt = salt_buf[0 .. salt_prefix.len + normalized_pass.len];

        // PBKDF2-HMAC-SHA512
        pbkdf2HmacSha512(
            normalized_phrase,
            salt,
            PBKDF2_ITERATIONS,
            &seed,
        );

        return seed;
    }
};
```

## Transaction Hints Bag

Manages hints for distributed signing (EIP-11)[^5][^6]:

```zig
const TransactionHintsBag = struct {
    /// Secret hints by input index (own commitments)
    secret_hints: std.AutoHashMap(usize, HintsBag),
    /// Public hints by input index (other signers' commitments)
    public_hints: std.AutoHashMap(usize, HintsBag),
    allocator: Allocator,

    pub fn empty(allocator: Allocator) TransactionHintsBag {
        return .{
            .secret_hints = std.AutoHashMap(usize, HintsBag).init(allocator),
            .public_hints = std.AutoHashMap(usize, HintsBag).init(allocator),
            .allocator = allocator,
        };
    }

    /// Replace all hints for an input index
    pub fn replaceHintsForInput(
        self: *TransactionHintsBag,
        index: usize,
        hints_bag: HintsBag,
    ) void {
        var secret_hints: []Hint = &.{};
        var public_hints: []Hint = &.{};

        for (hints_bag.hints) |hint| {
            switch (hint) {
                .own_commitment => secret_hints = append(secret_hints, hint),
                else => public_hints = append(public_hints, hint),
            }
        }

        self.secret_hints.put(index, HintsBag{ .hints = secret_hints }) catch {};
        self.public_hints.put(index, HintsBag{ .hints = public_hints }) catch {};
    }

    /// Add hints for an input (accumulate with existing)
    pub fn addHintsForInput(
        self: *TransactionHintsBag,
        index: usize,
        hints_bag: HintsBag,
    ) void {
        var existing_secret = self.secret_hints.get(index) orelse HintsBag.empty();
        var existing_public = self.public_hints.get(index) orelse HintsBag.empty();

        for (hints_bag.hints) |hint| {
            switch (hint) {
                .own_commitment => existing_secret.hints = append(existing_secret.hints, hint),
                else => existing_public.hints = append(existing_public.hints, hint),
            }
        }

        self.secret_hints.put(index, existing_secret) catch {};
        self.public_hints.put(index, existing_public) catch {};
    }

    /// Get all hints (secret + public) for an input
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

    /// Get only public hints (safe to share)
    pub fn publicHintsForInput(self: *const TransactionHintsBag, index: usize) HintsBag {
        return self.public_hints.get(index) orelse HintsBag.empty();
    }
};
```

## Distributed Signing Protocol (EIP-11)

```
Distributed Signing Flow
─────────────────────────────────────────────────────

Party A                              Party B
─────────                            ─────────

1. Generate Commitments
   commitmentsA = generateCommitments()
                                     commitmentsB = generateCommitments()

2. Exchange Public Hints
   publicA ──────────────────────────►
                    ◄────────────────── publicB

3. Sign with Combined Hints
   combinedA = commitmentsA + publicB
   partialSigA = sign(tx, combinedA)
                                     combinedB = commitmentsB + publicA
                                     partialSigB = sign(tx, combinedB)

4. Extract & Complete
   partialSigA ─────────────────────►
                                     extractedHints = extractHints(partialSigA)
                                     finalTx = sign(tx, commitmentsB + extracted)

Security: Secret hints (randomness r) NEVER leave their owner.
          Only public hints (commitments g^r) are exchanged.
```

```zig
/// Generate commitments for all transaction inputs
pub fn generateCommitments(
    wallet: *const Wallet,
    tx_context: *const TransactionContext(UnsignedTransaction),
    state_context: *const ErgoStateContext,
) !TransactionHintsBag {
    var public_keys: []SigmaBoolean = &.{};
    for (wallet.prover.secrets()) |secret| {
        public_keys = append(public_keys, secret.publicImage());
    }

    var hints_bag = TransactionHintsBag.empty(wallet.allocator);

    for (tx_context.spending_tx.inputs.items(), 0..) |_, idx| {
        const ctx = try makeContext(state_context, tx_context, idx);
        const input_box = tx_context.inputBoxes()[idx];

        // Reduce to SigmaBoolean
        const reduction = try reduceToCrypto(&input_box.ergo_tree, &ctx);

        // Generate commitments for propositions we can prove
        const input_hints = generateCommitmentsFor(
            &reduction.sigma_prop,
            public_keys,
        );
        hints_bag.addHintsForInput(idx, input_hints);
    }

    return hints_bag;
}

/// Extract hints from a partial signature
pub fn extractHints(
    tx: *const Transaction,
    real_propositions: []const SigmaBoolean,
    simulated_propositions: []const SigmaBoolean,
    boxes_to_spend: []const ErgoBox,
    data_boxes: []const ErgoBox,
) TransactionHintsBag {
    var hints_bag = TransactionHintsBag.empty(allocator);

    for (tx.inputs.items(), 0..) |input, idx| {
        const proof = input.spending_proof.proof;
        if (proof.isEmpty()) continue;

        const box = boxes_to_spend[idx];
        const extracted = extractHintsFromProof(
            &box.ergo_tree,
            proof.bytes(),
            real_propositions,
            simulated_propositions,
        );
        hints_bag.addHintsForInput(idx, extracted);
    }

    return hints_bag;
}
```

## Box Selection

Select inputs to satisfy target balance and tokens[^7][^8]:

```zig
const BoxSelector = struct {
    /// Selects boxes to satisfy target balance and tokens
    pub fn select(
        self: *const BoxSelector,
        inputs: []const ErgoBox,
        target_balance: BoxValue,
        target_tokens: []const Token,
    ) BoxSelectorError!BoxSelection {
        var selected: []ErgoBox = &.{};
        var total_value: u64 = 0;
        var total_tokens = std.AutoHashMap(TokenId, u64).init(allocator);
        defer total_tokens.deinit();

        // First pass: select boxes until targets met
        for (inputs) |box| {
            const needed = needsMoreBoxes(
                total_value,
                &total_tokens,
                target_balance.as_u64(),
                target_tokens,
            );
            if (!needed) break;

            selected = append(selected, box);
            total_value += box.value.as_u64();

            if (box.tokens) |tokens| {
                for (tokens.items()) |token| {
                    const entry = total_tokens.getOrPut(token.token_id);
                    if (entry.found_existing) {
                        entry.value_ptr.* += token.amount.value;
                    } else {
                        entry.value_ptr.* = token.amount.value;
                    }
                }
            }
        }

        // Check if targets met
        if (total_value < target_balance.as_u64()) {
            return error.NotEnoughCoins;
        }

        for (target_tokens) |target| {
            const have = total_tokens.get(target.token_id) orelse 0;
            if (have < target.amount.value) {
                return error.NotEnoughTokens;
            }
        }

        // Calculate change
        const change = calculateChange(
            total_value,
            &total_tokens,
            target_balance.as_u64(),
            target_tokens,
        );

        return BoxSelection{
            .boxes = try BoundedVec(ErgoBox, 1, MAX_INPUTS).fromSlice(selected),
            .change_boxes = change,
        };
    }
};

const BoxSelection = struct {
    /// Selected boxes to spend
    boxes: BoundedVec(ErgoBox, 1, MAX_INPUTS),
    /// Change boxes to create
    change_boxes: []ErgoBoxAssetsData,
};

const BoxSelectorError = error{
    NotEnoughCoins,
    NotEnoughTokens,
    TokenAmountError,
    NotEnoughCoinsForChangeBox,
    SelectedInputsOutOfBounds,
};
```

## Transaction Signing Flow

```
Transaction Signing Flow
─────────────────────────────────────────────────────

┌──────────────────────────────────────────────────┐
│ 1. User Request                                  │
│    ├── Target balance                            │
│    └── Target tokens                             │
└──────────────────────────┬───────────────────────┘
                           ▼
┌──────────────────────────────────────────────────┐
│ 2. Box Selection                                 │
│    ├── BoxSelector.select(inputs, target)        │
│    └── Returns: boxes + change                   │
└──────────────────────────┬───────────────────────┘
                           ▼
┌──────────────────────────────────────────────────┐
│ 3. Build Unsigned Transaction                    │
│    ├── inputs: selected boxes                    │
│    ├── data_inputs: read-only references         │
│    └── output_candidates: targets + change       │
└──────────────────────────┬───────────────────────┘
                           ▼
┌──────────────────────────────────────────────────┐
│ 4. Sign Transaction                              │
│    For each input:                               │
│    ├── Create Context                            │
│    ├── Get hints for input                       │
│    ├── prover.prove(tree, ctx, message, hints)   │
│    └── Accumulate cost                           │
└──────────────────────────┬───────────────────────┘
                           ▼
┌──────────────────────────────────────────────────┐
│ 5. Signed Transaction                            │
│    └── Submit to mempool                         │
└──────────────────────────────────────────────────┘
```

```zig
/// Sign transaction with prover
pub fn signTransaction(
    prover: *const Prover,
    tx_context: *const TransactionContext(UnsignedTransaction),
    state_context: *const ErgoStateContext,
    tx_hints: ?*const TransactionHintsBag,
) !Transaction {
    const tx = tx_context.spending_tx;
    const message = try tx.bytesToSign();

    var signed_inputs: []Input = &.{};

    for (tx.inputs.items(), 0..) |unsigned_input, idx| {
        const ctx = try makeContext(state_context, tx_context, idx);

        // Get hints for this input
        const hints = if (tx_hints) |h| h.allHintsForInput(idx) else HintsBag.empty();

        const input_box = tx_context.getInputBox(unsigned_input.box_id) orelse
            return error.InputBoxNotFound;

        // Generate proof
        const prover_result = try prover.prove(
            &input_box.ergo_tree,
            &ctx,
            message,
            &hints,
        );

        signed_inputs = append(signed_inputs, Input{
            .box_id = unsigned_input.box_id,
            .spending_proof = prover_result,
        });
    }

    return Transaction.new(
        try TxIoVec(Input).fromSlice(signed_inputs),
        tx.data_inputs,
        tx.output_candidates,
    );
}
```

## Asset Extraction

Calculate token access costs[^9]:

```zig
const ErgoBoxAssetExtractor = struct {
    pub const MAX_ASSETS_PER_BOX: usize = 255;

    /// Extract total token amounts from boxes
    pub fn extractAssets(
        boxes: []const ErgoBoxCandidate,
    ) !struct { assets: std.AutoHashMap(TokenId, u64), count: usize } {
        var assets = std.AutoHashMap(TokenId, u64).init(allocator);
        var total_count: usize = 0;

        for (boxes) |box| {
            if (box.tokens) |tokens| {
                if (tokens.len() > MAX_ASSETS_PER_BOX) {
                    return error.TooManyAssetsInBox;
                }

                for (tokens.items()) |token| {
                    const entry = assets.getOrPut(token.token_id);
                    if (entry.found_existing) {
                        entry.value_ptr.* = std.math.add(
                            u64,
                            entry.value_ptr.*,
                            token.amount.value,
                        ) catch return error.Overflow;
                    } else {
                        entry.value_ptr.* = token.amount.value;
                    }
                }
                total_count += tokens.len();
            }
        }

        return .{ .assets = assets, .count = total_count };
    }

    /// Calculate total token access cost
    pub fn totalAssetsAccessCost(
        in_assets_num: usize,
        in_assets_size: usize,
        out_assets_num: usize,
        out_assets_size: usize,
        token_access_cost: u32,
    ) u64 {
        // Cost to iterate through all tokens
        const all_assets_cost = (out_assets_num + in_assets_num) * token_access_cost;
        // Cost to check preservation of unique tokens
        const unique_assets_cost = (in_assets_size + out_assets_size) * token_access_cost;
        return all_assets_cost + unique_assets_cost;
    }
};
```

## Wallet Errors

```zig
const WalletError = error{
    /// Transaction signing failed
    TxSigningError,
    /// Prover failed to generate proof
    ProverError,
    /// Key derivation failed
    ExtSecretKeyError,
    /// Secret key parsing failed
    SecretKeyParsingError,
    /// Wallet not initialized
    WalletNotInitialized,
    /// Wallet locked
    WalletLocked,
    /// Wallet already unlocked
    WalletAlreadyUnlocked,
    /// Box selection failed
    BoxSelectionError,
};
```

## Distributed Signing Example

```zig
// Party A: Generate commitments
const commitments_a = try wallet_a.generateCommitments(&tx_context, &state_context);

// Party B: Generate commitments
const commitments_b = try wallet_b.generateCommitments(&tx_context, &state_context);

// Exchange public hints (safe to share)
const public_a = commitments_a.publicHintsForInput(0);
const public_b = commitments_b.publicHintsForInput(0);

// Party A: Sign with combined hints
var combined_a = commitments_a;
combined_a.addHintsForInput(0, public_b);
const partial_sig_a = try wallet_a.signTransaction(&tx_context, &state_context, &combined_a);

// Party B: Extract hints from A's partial signature
const extracted = extractHints(
    &partial_sig_a,
    real_propositions,
    simulated_propositions,
    boxes_to_spend,
    data_boxes,
);

// Party B: Complete signing
var final_hints = commitments_b;
final_hints.addHintsForInput(0, extracted.allHintsForInput(0));
const final_tx = try wallet_b.signTransaction(&tx_context, &state_context, &final_hints);
```

## Summary

- **Wallet** wraps prover with high-level signing API
- **Mnemonic** converts BIP-39 phrase to seed via PBKDF2
- **TransactionHintsBag** separates secret/public hints for distributed signing
- **BoxSelector** finds optimal input set for target balance/tokens
- **Distributed signing** (EIP-11) exchanges commitments, never secrets
- **Asset extraction** calculates token access costs

---

*Next: [Chapter 27: High-Level SDK](../part9/ch27-high-level-sdk.md)*

[^1]: Scala: [`ErgoWalletService.scala:36-61`](https://github.com/ergoplatform/ergo/blob/master/ergo/src/main/scala/org/ergoplatform/nodeView/wallet/ErgoWalletService.scala#L36-L61)

[^2]: Rust: [`wallet.rs:52-93`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergo-lib/src/wallet.rs#L52-L93)

[^3]: Scala: [`Mnemonic.scala`](https://github.com/ergoplatform/ergo/blob/master/ergo-wallet/src/main/scala/org/ergoplatform/wallet/mnemonic/Mnemonic.scala)

[^4]: Rust: [`mnemonic.rs:20-37`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergo-lib/src/wallet/mnemonic.rs#L20-L37)

[^5]: Scala: [`TransactionHintsBag.scala:5-56`](https://github.com/ergoplatform/ergo/blob/master/ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/TransactionHintsBag.scala#L5-L56)

[^6]: Rust: [`wallet.rs:259-347`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergo-lib/src/wallet.rs#L259-L347)

[^7]: Scala: [`BoxSelector.scala`](https://github.com/ergoplatform/ergo/blob/master/ergo-wallet/src/main/scala/org/ergoplatform/wallet/boxes/BoxSelector.scala)

[^8]: Rust: [`box_selector.rs:34-46`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergo-lib/src/wallet/box_selector.rs#L34-L46)

[^9]: Scala: [`ErgoBoxAssetExtractor.scala:55-65`](https://github.com/ergoplatform/ergo/blob/master/ergo-wallet/src/main/scala/org/ergoplatform/wallet/boxes/ErgoBoxAssetExtractor.scala#L55-L65)
