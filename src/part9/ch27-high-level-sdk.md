# Chapter 27: High-Level SDK

## Prerequisites

- Transaction validation ([Chapter 24](../part8/ch24-transaction-validation.md))
- Prover implementation ([Chapter 15](../part5/ch15-prover-implementation.md))
- Wallet signing ([Chapter 26](../part8/ch26-wallet-signing.md))

## Learning Objectives

- Understand SDK architecture for transaction building
- Implement TxBuilder with builder pattern
- Master the reduce-then-sign pipeline
- Work with TransactionContext and BoxSelection

## SDK Architecture

The SDK provides a layered abstraction from low-level cryptography to high-level transaction building[^1][^2]:

```
SDK Layer Architecture
══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│                     Application Layer                           │
│   TxBuilder    BoxSelector    ErgoBoxCandidateBuilder          │
├────────────────────────────────────────────────────────────────┤
│                     Wallet Layer                                │
│   Wallet    TransactionContext    TransactionHintsBag          │
├────────────────────────────────────────────────────────────────┤
│                     Reduction Layer                             │
│   reduce_tx()    ReducedTransaction    ReducedInput            │
├────────────────────────────────────────────────────────────────┤
│                     Signing Layer                               │
│   sign_transaction()    sign_reduced_transaction()             │
├────────────────────────────────────────────────────────────────┤
│                     Interpreter Layer                           │
│   Prover    Verifier    reduce_to_crypto()                     │
└────────────────────────────────────────────────────────────────┘
```

## Transaction Builder

The builder pattern constructs unsigned transactions with validation[^3][^4]:

```zig
const TxBuilder = struct {
    box_selection: BoxSelection,
    data_inputs: std.ArrayList(DataInput),
    output_candidates: std.ArrayList(ErgoBoxCandidate),
    current_height: u32,
    fee_amount: BoxValue,
    change_address: Address,
    context_extensions: std.AutoHashMap(BoxId, ContextExtension),
    token_burn_permit: std.ArrayList(Token),
    allocator: Allocator,

    pub fn init(
        box_selection: BoxSelection,
        output_candidates: []const ErgoBoxCandidate,
        current_height: u32,
        fee_amount: BoxValue,
        change_address: Address,
        allocator: Allocator,
    ) !TxBuilder {
        var outputs = std.ArrayList(ErgoBoxCandidate).init(allocator);
        try outputs.appendSlice(output_candidates);

        return .{
            .box_selection = box_selection,
            .data_inputs = std.ArrayList(DataInput).init(allocator),
            .output_candidates = outputs,
            .current_height = current_height,
            .fee_amount = fee_amount,
            .change_address = change_address,
            .context_extensions = std.AutoHashMap(BoxId, ContextExtension).init(allocator),
            .token_burn_permit = std.ArrayList(Token).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *TxBuilder) void {
        self.data_inputs.deinit();
        self.output_candidates.deinit();
        self.context_extensions.deinit();
        self.token_burn_permit.deinit();
    }

    pub fn setDataInputs(self: *TxBuilder, data_inputs: []const DataInput) !void {
        self.data_inputs.clearRetainingCapacity();
        try self.data_inputs.appendSlice(data_inputs);
    }

    pub fn setContextExtension(self: *TxBuilder, box_id: BoxId, ext: ContextExtension) !void {
        try self.context_extensions.put(box_id, ext);
    }

    pub fn setTokenBurnPermit(self: *TxBuilder, tokens: []const Token) !void {
        self.token_burn_permit.clearRetainingCapacity();
        try self.token_burn_permit.appendSlice(tokens);
    }
};
```

## Build Validation

Building performs comprehensive validation before creating the transaction[^5][^6]:

```zig
pub fn build(self: *TxBuilder) !UnsignedTransaction {
    // Validate inputs
    if (self.box_selection.boxes.items.len == 0) {
        return error.EmptyInputs;
    }
    if (self.output_candidates.items.len == 0) {
        return error.EmptyOutputs;
    }
    if (self.box_selection.boxes.items.len > std.math.maxInt(u16)) {
        return error.TooManyInputs;
    }

    // Check for duplicate inputs
    var seen = std.AutoHashMap(BoxId, void).init(self.allocator);
    defer seen.deinit();
    for (self.box_selection.boxes.items) |box| {
        const result = try seen.getOrPut(box.box_id);
        if (result.found_existing) {
            return error.DuplicateInputs;
        }
    }

    // Build output candidates with change boxes
    var all_outputs = try self.buildOutputCandidates();
    defer all_outputs.deinit();

    // Validate coin preservation
    const total_in = sumValue(self.box_selection.boxes.items);
    const total_out = sumValue(all_outputs.items);

    if (total_out > total_in) {
        return error.NotEnoughCoinsInInputs;
    }
    if (total_out < total_in) {
        return error.NotEnoughCoinsInOutputs;
    }

    // Validate token balance
    try self.validateTokenBalance(all_outputs.items);

    // Create unsigned inputs with context extensions
    var unsigned_inputs = std.ArrayList(UnsignedInput).init(self.allocator);
    for (self.box_selection.boxes.items) |box| {
        const ext = self.context_extensions.get(box.box_id) orelse
            ContextExtension.empty();
        try unsigned_inputs.append(.{
            .box_id = box.box_id,
            .extension = ext,
        });
    }

    return UnsignedTransaction{
        .inputs = try unsigned_inputs.toOwnedSlice(),
        .data_inputs = try self.data_inputs.toOwnedSlice(),
        .output_candidates = try all_outputs.toOwnedSlice(),
    };
}

fn buildOutputCandidates(self: *TxBuilder) !std.ArrayList(ErgoBoxCandidate) {
    var outputs = std.ArrayList(ErgoBoxCandidate).init(self.allocator);

    // Add user-specified outputs
    try outputs.appendSlice(self.output_candidates.items);

    // Add change boxes from selection
    const change_tree = try Contract.payToAddress(self.change_address);
    for (self.box_selection.change_boxes.items) |change| {
        var candidate = try ErgoBoxCandidateBuilder.init(
            change.value,
            change_tree,
            self.current_height,
            self.allocator,
        );
        for (change.tokens) |token| {
            try candidate.addToken(token);
        }
        try outputs.append(try candidate.build());
    }

    // Add miner fee box
    const fee_box = try newMinerFeeBox(self.fee_amount, self.current_height);
    try outputs.append(fee_box);

    return outputs;
}
```

## Token Balance Validation

Token flow must be explicitly validated[^7][^8]:

```zig
fn validateTokenBalance(self: *TxBuilder, outputs: []const ErgoBoxCandidate) !void {
    const input_tokens = try sumTokens(self.box_selection.boxes.items, self.allocator);
    defer input_tokens.deinit();

    const output_tokens = try sumTokens(outputs, self.allocator);
    defer output_tokens.deinit();

    // First input's box ID can be minted as new token
    const first_input_id = TokenId.fromBoxId(self.box_selection.boxes.items[0].box_id);

    // Filter out minted token from outputs
    var minted_count: usize = 0;
    var output_without_minted = std.AutoHashMap(TokenId, TokenAmount).init(self.allocator);
    defer output_without_minted.deinit();

    var iter = output_tokens.iterator();
    while (iter.next()) |entry| {
        if (entry.key_ptr.*.eql(first_input_id)) {
            minted_count += 1;
        } else {
            try output_without_minted.put(entry.key_ptr.*, entry.value_ptr.*);
        }
    }

    // Can only mint one token per transaction
    if (minted_count > 1) {
        return error.CannotMintMultipleTokens;
    }

    // Check all output tokens exist in inputs
    var out_iter = output_without_minted.iterator();
    while (out_iter.next()) |entry| {
        const input_amt = input_tokens.get(entry.key_ptr.*) orelse {
            return error.NotEnoughTokens;
        };
        if (input_amt < entry.value_ptr.*) {
            return error.NotEnoughTokens;
        }
    }

    // Check token burn permits
    const burned = try subtractTokens(input_tokens, output_without_minted, self.allocator);
    defer burned.deinit();

    try self.checkBurnPermit(burned);
}

fn checkBurnPermit(self: *TxBuilder, burned: std.AutoHashMap(TokenId, TokenAmount)) !void {
    // Build permit map
    var permits = std.AutoHashMap(TokenId, TokenAmount).init(self.allocator);
    defer permits.deinit();
    for (self.token_burn_permit.items) |token| {
        try permits.put(token.id, token.amount);
    }

    // Every burned token must have permit
    var iter = burned.iterator();
    while (iter.next()) |entry| {
        const permit_amt = permits.get(entry.key_ptr.*) orelse {
            return error.TokenBurnPermitMissing;
        };
        if (entry.value_ptr.* > permit_amt) {
            return error.TokenBurnPermitExceeded;
        }
    }

    // Every permit must be used exactly
    var permit_iter = permits.iterator();
    while (permit_iter.next()) |entry| {
        const burned_amt = burned.get(entry.key_ptr.*) orelse {
            return error.TokenBurnPermitUnused;
        };
        if (burned_amt < entry.value_ptr.*) {
            return error.TokenBurnPermitUnused;
        }
    }
}
```

## Box Candidate Builder

Constructs output boxes with fluent API:

```zig
const ErgoBoxCandidateBuilder = struct {
    value: BoxValue,
    ergo_tree: ErgoTree,
    creation_height: u32,
    tokens: std.ArrayList(Token),
    registers: [6]?Constant, // R4-R9
    allocator: Allocator,

    pub fn init(
        value: BoxValue,
        ergo_tree: ErgoTree,
        creation_height: u32,
        allocator: Allocator,
    ) !ErgoBoxCandidateBuilder {
        return .{
            .value = value,
            .ergo_tree = ergo_tree,
            .creation_height = creation_height,
            .tokens = std.ArrayList(Token).init(allocator),
            .registers = [_]?Constant{null} ** 6,
            .allocator = allocator,
        };
    }

    pub fn addToken(self: *ErgoBoxCandidateBuilder, token: Token) !void {
        if (self.tokens.items.len >= MAX_TOKENS) {
            return error.TooManyTokens;
        }
        try self.tokens.append(token);
    }

    pub fn mintToken(
        self: *ErgoBoxCandidateBuilder,
        token: Token,
        name: []const u8,
        description: []const u8,
        decimals: u8,
    ) !void {
        try self.addToken(token);
        // Store metadata in R4-R6
        self.registers[0] = Constant.fromBytes(name);
        self.registers[1] = Constant.fromBytes(description);
        self.registers[2] = Constant.fromByte(decimals);
    }

    pub fn setRegister(self: *ErgoBoxCandidateBuilder, reg: RegisterId, value: Constant) void {
        const idx = @intFromEnum(reg) - 4; // R4 = 0, R5 = 1, etc.
        self.registers[idx] = value;
    }

    pub fn build(self: *ErgoBoxCandidateBuilder) !ErgoBoxCandidate {
        return ErgoBoxCandidate{
            .value = self.value,
            .ergo_tree = self.ergo_tree,
            .creation_height = self.creation_height,
            .tokens = try self.tokens.toOwnedSlice(),
            .additional_registers = self.registers,
        };
    }
};
```

## Transaction Context

Bundles transaction with input boxes for signing[^9][^10]:

```zig
const TransactionContext = struct {
    spending_tx: UnsignedTransaction,
    input_boxes: []const ErgoBox,
    data_boxes: ?[]const ErgoBox,

    pub fn init(
        spending_tx: UnsignedTransaction,
        input_boxes: []const ErgoBox,
        data_boxes: ?[]const ErgoBox,
    ) !TransactionContext {
        // Validate input boxes match transaction inputs
        if (input_boxes.len != spending_tx.inputs.len) {
            return error.InputBoxCountMismatch;
        }

        for (spending_tx.inputs, input_boxes) |input, box| {
            if (!input.box_id.eql(box.box_id())) {
                return error.InputBoxIdMismatch;
            }
        }

        // Validate data boxes if present
        if (spending_tx.data_inputs) |data_inputs| {
            const data = data_boxes orelse return error.DataInputBoxNotFound;
            if (data.len != data_inputs.len) {
                return error.DataInputBoxCountMismatch;
            }
        }

        return .{
            .spending_tx = spending_tx,
            .input_boxes = input_boxes,
            .data_boxes = data_boxes,
        };
    }

    pub fn getInputBox(self: *const TransactionContext, box_id: BoxId) ?*const ErgoBox {
        for (self.input_boxes) |*box| {
            if (box.box_id().eql(box_id)) {
                return box;
            }
        }
        return null;
    }
};
```

## Box Selection

Selects input boxes to satisfy output requirements[^11][^12]:

```zig
const BoxSelection = struct {
    boxes: std.ArrayList(ErgoBox),
    change_boxes: std.ArrayList(ErgoBoxAssets),

    const ErgoBoxAssets = struct {
        value: BoxValue,
        tokens: []const Token,
    };
};

const SimpleBoxSelector = struct {
    pub fn select(
        available: []const ErgoBox,
        target_value: BoxValue,
        target_tokens: []const Token,
        allocator: Allocator,
    ) !BoxSelection {
        var selected = std.ArrayList(ErgoBox).init(allocator);
        var total_value: u64 = 0;
        var token_sums = std.AutoHashMap(TokenId, TokenAmount).init(allocator);
        defer token_sums.deinit();

        // Greedy selection
        for (available) |box| {
            const needed = checkNeed(total_value, target_value, token_sums, target_tokens);
            if (!needed) break;

            try selected.append(box);
            total_value += box.value.as_u64();

            for (box.tokens) |token| {
                const entry = try token_sums.getOrPut(token.id);
                if (entry.found_existing) {
                    entry.value_ptr.* = try entry.value_ptr.*.checkedAdd(token.amount);
                } else {
                    entry.value_ptr.* = token.amount;
                }
            }
        }

        // Calculate change
        var change_boxes = std.ArrayList(BoxSelection.ErgoBoxAssets).init(allocator);
        const change_value = total_value - target_value.as_u64();
        if (change_value > 0) {
            const change_tokens = try calculateChangeTokens(token_sums, target_tokens, allocator);
            try change_boxes.append(.{
                .value = BoxValue.init(change_value) catch return error.ChangeValueTooSmall,
                .tokens = change_tokens,
            });
        }

        return .{
            .boxes = selected,
            .change_boxes = change_boxes,
        };
    }
};
```

## Reduced Transaction

Script reduction separates evaluation from signing[^13][^14]:

```zig
const ReducedInput = struct {
    sigma_prop: SigmaBoolean,
    cost: u64,
    extension: ContextExtension,
};

const ReducedTransaction = struct {
    unsigned_tx: UnsignedTransaction,
    reduced_inputs: []const ReducedInput,
    tx_cost: u32,

    pub fn reducedInputs(self: *const ReducedTransaction) []const ReducedInput {
        return self.reduced_inputs;
    }
};

/// Reduce transaction inputs to sigma propositions
pub fn reduceTx(
    tx_context: TransactionContext,
    state_context: *const ErgoStateContext,
    allocator: Allocator,
) !ReducedTransaction {
    var reduced_inputs = std.ArrayList(ReducedInput).init(allocator);

    for (tx_context.spending_tx.inputs, 0..) |input, idx| {
        // Build evaluation context
        var ctx = try makeContext(state_context, &tx_context, idx);

        // Get input box
        const input_box = tx_context.getInputBox(input.box_id) orelse
            return error.InputBoxNotFound;

        // Reduce ErgoTree to SigmaBoolean
        const result = try reduceToCrypto(&input_box.ergo_tree, &ctx);

        try reduced_inputs.append(.{
            .sigma_prop = result.sigma_prop,
            .cost = result.cost,
            .extension = input.extension,
        });
    }

    return .{
        .unsigned_tx = tx_context.spending_tx,
        .reduced_inputs = try reduced_inputs.toOwnedSlice(),
        .tx_cost = 0,
    };
}
```

## Signing Pipeline

```
Signing Flow
══════════════════════════════════════════════════════════════════

┌─────────────────┐     ┌──────────────────┐     ┌───────────────┐
│ UnsignedTx      │     │ ReducedTx        │     │ SignedTx      │
│ + InputBoxes    │────▶│ (SigmaProps)     │────▶│ (Proofs)      │
│ + StateContext  │     │                  │     │               │
└─────────────────┘     └──────────────────┘     └───────────────┘
        │                       │                       │
        │  reduce_tx()          │  sign_reduced_tx()    │
        │  (needs context)      │  (context-free)       │
        ▼                       ▼                       ▼
   ┌─────────┐            ┌─────────┐            ┌─────────┐
   │ Online  │            │ Offline │            │ Verify  │
   │ Wallet  │            │ Wallet  │            │ Node    │
   └─────────┘            └─────────┘            └─────────┘
```

Transaction signing with optional hints[^15][^16]:

```zig
pub fn signTransaction(
    prover: *const Prover,
    tx_context: TransactionContext,
    state_context: *const ErgoStateContext,
    tx_hints: ?*const TransactionHintsBag,
) !Transaction {
    const message = try tx_context.spending_tx.bytesToSign();

    var signed_inputs = std.ArrayList(Input).init(prover.allocator);
    for (tx_context.spending_tx.inputs, 0..) |input, idx| {
        const signed = try signTxInput(
            prover,
            &tx_context,
            state_context,
            tx_hints,
            idx,
            message,
        );
        try signed_inputs.append(signed);
    }

    return Transaction{
        .inputs = try signed_inputs.toOwnedSlice(),
        .data_inputs = tx_context.spending_tx.data_inputs,
        .outputs = tx_context.spending_tx.output_candidates,
    };
}

pub fn signReducedTransaction(
    prover: *const Prover,
    reduced_tx: ReducedTransaction,
    tx_hints: ?*const TransactionHintsBag,
) !Transaction {
    const message = try reduced_tx.unsigned_tx.bytesToSign();

    var signed_inputs = std.ArrayList(Input).init(prover.allocator);
    for (reduced_tx.unsigned_tx.inputs, 0..) |input, idx| {
        const reduced_input = reduced_tx.reduced_inputs[idx];

        // Get hints for this input
        const hints = if (tx_hints) |bag|
            bag.allHintsForInput(idx)
        else
            HintsBag.empty();

        // Generate proof from sigma proposition
        const proof = try prover.generateProof(
            reduced_input.sigma_prop,
            message,
            &hints,
        );

        try signed_inputs.append(.{
            .box_id = input.box_id,
            .spending_proof = .{
                .proof = proof,
                .extension = reduced_input.extension,
            },
        });
    }

    return Transaction{
        .inputs = try signed_inputs.toOwnedSlice(),
        .data_inputs = reduced_tx.unsigned_tx.data_inputs,
        .outputs = reduced_tx.unsigned_tx.output_candidates,
    };
}
```

## Miner Fee Box

Standard miner fee output:

```zig
/// Miner fee ErgoTree (false proposition with height constraint)
const MINERS_FEE_ERGO_TREE = [_]u8{
    0x10, 0x05, 0x04, 0x00, 0x04, 0x00, 0x0e, 0x36,
    0x10, 0x02, 0x04, 0xa0, 0x0b, 0x08, 0xcd, 0x02,
    // ... (standard miner fee script)
};

pub fn newMinerFeeBox(fee: BoxValue, creation_height: u32) !ErgoBoxCandidate {
    const tree = try ErgoTree.sigmaParse(&MINERS_FEE_ERGO_TREE);

    return ErgoBoxCandidate{
        .value = fee,
        .ergo_tree = tree,
        .creation_height = creation_height,
        .tokens = &[_]Token{},
        .additional_registers = [_]?Constant{null} ** 6,
    };
}

/// Suggested transaction fee (1.1 mERG)
pub const SUGGESTED_TX_FEE = BoxValue.init(1_100_000) catch unreachable;
```

## Reduced Transaction Serialization

EIP-19 format for cold wallet transfer[^17][^18]:

```zig
const ReducedTransactionSerializer = struct {
    pub fn serialize(tx: *const ReducedTransaction, writer: anytype) !void {
        // Write message to sign (includes all tx data)
        const msg = try tx.unsigned_tx.bytesToSign();
        try writer.writeInt(u32, @intCast(msg.len), .little);
        try writer.writeAll(msg);

        // Write reduced inputs
        for (tx.reduced_inputs) |red_in| {
            try SigmaBoolean.serialize(&red_in.sigma_prop, writer);
            try writer.writeInt(u64, red_in.cost, .little);
        }

        try writer.writeInt(u32, tx.tx_cost, .little);
    }

    pub fn parse(reader: anytype, allocator: Allocator) !ReducedTransaction {
        // Read and parse message
        const msg_len = try reader.readInt(u32, .little);
        const msg = try allocator.alloc(u8, msg_len);
        try reader.readNoEof(msg);

        const tx = try Transaction.sigmaParse(msg);

        // Read reduced inputs
        var reduced_inputs = std.ArrayList(ReducedInput).init(allocator);
        for (tx.inputs) |input| {
            const sigma_prop = try SigmaBoolean.parse(reader);
            const cost = try reader.readInt(u64, .little);

            try reduced_inputs.append(.{
                .sigma_prop = sigma_prop,
                .cost = cost,
                .extension = input.spending_proof.extension,
            });
        }

        const tx_cost = try reader.readInt(u32, .little);

        return .{
            .unsigned_tx = tx.toUnsigned(),
            .reduced_inputs = try reduced_inputs.toOwnedSlice(),
            .tx_cost = tx_cost,
        };
    }
};
```

## Cold Wallet Flow

```
Cold Wallet Signing
══════════════════════════════════════════════════════════════════

Online Wallet (Hot)              Cold Wallet (Air-gapped)
──────────────────────           ────────────────────────
       │                                    │
  Build Unsigned Tx                         │
       │                                    │
  reduce_tx()                               │
       │                                    │
  Serialize ReducedTx ─────────────────────▶│
  (QR code / USB)                           │
       │                               Parse ReducedTx
       │                                    │
       │                               sign_reduced_tx()
       │                               (uses secrets)
       │                                    │
       │◀──────────────────────── Serialize SignedTx
       │                          (QR code / USB)
  Broadcast Tx                              │
       │                                    │
       ▼                                    ▼
```

## Complete Usage Example

```zig
pub fn buildAndSignTransaction(
    wallet: *const Wallet,
    available_boxes: []const ErgoBox,
    recipient: Address,
    amount: u64,
    state_context: *const ErgoStateContext,
    allocator: Allocator,
) !Transaction {
    const current_height = state_context.pre_header.height;

    // 1. Build output
    const recipient_tree = try Contract.payToAddress(recipient);
    var out_builder = try ErgoBoxCandidateBuilder.init(
        try BoxValue.init(amount),
        recipient_tree,
        current_height,
        allocator,
    );
    const output = try out_builder.build();

    // 2. Select inputs
    const total_needed = try BoxValue.init(amount + SUGGESTED_TX_FEE.as_u64());
    const selection = try SimpleBoxSelector.select(
        available_boxes,
        total_needed,
        &[_]Token{},
        allocator,
    );

    // 3. Build transaction
    const change_address = wallet.getP2PKAddress();
    var builder = try TxBuilder.init(
        selection,
        &[_]ErgoBoxCandidate{output},
        current_height,
        SUGGESTED_TX_FEE,
        change_address,
        allocator,
    );
    defer builder.deinit();

    const unsigned_tx = try builder.build();

    // 4. Create transaction context
    const tx_context = try TransactionContext.init(
        unsigned_tx,
        selection.boxes.items,
        null,
    );

    // 5. Sign transaction
    return wallet.signTransaction(tx_context, state_context, null);
}
```

## Summary

- **TxBuilder** constructs unsigned transactions with validation
- **BoxSelection** satisfies value and token requirements
- **ErgoBoxCandidateBuilder** creates output boxes with fluent API
- **TransactionContext** bundles transaction with input data
- **reduce_tx()** separates script evaluation from signing
- **ReducedTransaction** enables air-gapped cold wallet signing
- **Token burn** requires explicit permits to prevent accidents

---

*Next: [Chapter 28: Key Derivation](./ch28-key-derivation.md)*

[^1]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/`

[^2]: Rust: `ergo-lib/src/wallet.rs:52-244`

[^3]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/UnsignedTransactionBuilder.scala`

[^4]: Rust: `ergo-lib/src/wallet/tx_builder.rs:41-78`

[^5]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/UnsignedTransactionBuilder.scala:79-111`

[^6]: Rust: `ergo-lib/src/wallet/tx_builder.rs:144-258`

[^7]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/AppkitProvingInterpreter.scala` (token validation)

[^8]: Rust: `ergo-lib/src/wallet/tx_builder.rs:214-243`

[^9]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/Transactions.scala:17-46`

[^10]: Rust: `ergo-lib/src/wallet/tx_context.rs`

[^11]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/BoxSelectionResult.scala`

[^12]: Rust: `ergo-lib/src/wallet/box_selector.rs`

[^13]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/AppkitProvingInterpreter.scala:274-289`

[^14]: Rust: `ergo-lib/src/chain/transaction/reduced.rs:25-67`

[^15]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/AppkitProvingInterpreter.scala:81-95`

[^16]: Rust: `ergo-lib/src/wallet/signing.rs:143-168`

[^17]: Scala: `sdk/shared/src/main/scala/org/ergoplatform/sdk/AppkitProvingInterpreter.scala:292-336`

[^18]: Rust: `ergo-lib/src/chain/transaction/reduced.rs:108-154`
