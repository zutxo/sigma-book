# Chapter 24: Transaction Validation

## Prerequisites

- Interpreter architecture ([Chapter 14](../part5/ch14-sigma-interpreter-architecture.md))
- Box model ([Chapter 22](../part7/ch22-box-model.md))
- Interpreter wrappers ([Chapter 23](./ch23-interpreter-wrappers.md))

## Learning Objectives

- Understand the two-phase validation pipeline
- Master stateless validation rules
- Learn stateful validation with cost accumulation
- Implement asset preservation checks

## Validation Pipeline

Transaction validation occurs in two phases[^1][^2]:

```
Transaction Validation Pipeline
─────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────┐
│           STATELESS VALIDATION                      │
│   (No blockchain state required)                    │
├─────────────────────────────────────────────────────┤
│  • Has inputs?         (at least 1)                 │
│  • Has outputs?        (at least 1)                 │
│  • Count limits        (≤ 32,767 each)              │
│  • No negative values  (outputs ≥ 0)                │
│  • Output sum valid    (no overflow)                │
│  • Unique inputs       (no double-spend)            │
└──────────────────────────┬──────────────────────────┘
                           │ Pass
                           ▼
┌─────────────────────────────────────────────────────┐
│           STATEFUL VALIDATION                       │
│   (Requires UTXO state and blockchain context)      │
├─────────────────────────────────────────────────────┤
│  1. Calculate initial cost                          │
│  2. Verify outputs (dust, height, size)             │
│  3. Check ERG preservation                          │
│  4. Verify asset preservation                       │
│  5. Verify input scripts (accumulate cost)          │
│  6. Check re-emission rules (EIP-27)                │
└─────────────────────────────────────────────────────┘
```

## Transaction Structure

```zig
const Transaction = struct {
    /// Transaction ID (Blake2b256 of serialized tx without proofs)
    tx_id: TxId,
    /// Input boxes to spend (with proofs)
    inputs: TxIoVec(Input),
    /// Read-only input references (no proofs)
    data_inputs: ?TxIoVec(DataInput),
    /// Output box candidates
    output_candidates: TxIoVec(ErgoBoxCandidate),
    /// Materialized outputs (with tx_id and index)
    outputs: TxIoVec(ErgoBox),

    pub const MAX_OUTPUTS_COUNT: usize = std.math.maxInt(u16);

    pub fn init(
        inputs: TxIoVec(Input),
        data_inputs: ?TxIoVec(DataInput),
        output_candidates: TxIoVec(ErgoBoxCandidate),
    ) !Transaction {
        // First pass: compute outputs with zero tx_id
        const zero_outputs = try output_candidates.mapIndexed(
            struct {
                fn f(idx: usize, bc: *const ErgoBoxCandidate) !ErgoBox {
                    return ErgoBox.fromBoxCandidate(bc, TxId.zero(), @intCast(idx));
                }
            }.f,
        );

        var tx = Transaction{
            .tx_id = TxId.zero(),
            .inputs = inputs,
            .data_inputs = data_inputs,
            .output_candidates = output_candidates,
            .outputs = zero_outputs,
        };

        // Compute actual tx_id
        tx.tx_id = try tx.calcTxId();

        // Update outputs with correct tx_id
        tx.outputs = try output_candidates.mapIndexed(
            struct {
                fn f(idx: usize, bc: *const ErgoBoxCandidate) !ErgoBox {
                    return ErgoBox.fromBoxCandidate(bc, tx.tx_id, @intCast(idx));
                }
            }.f,
        );

        return tx;
    }
};
```

## Validation Error Types

```zig
const TxValidationError = error{
    /// Output ERG sum overflow
    OutputSumOverflow,
    /// Input ERG sum overflow
    InputSumOverflow,
    /// Same box spent twice
    DoubleSpend,
    /// ERG not preserved (inputs != outputs)
    ErgPreservationError,
    /// Token amounts not preserved
    TokenPreservationError,
    /// Output below dust threshold
    DustOutput,
    /// Creation height > current height
    InvalidHeightError,
    /// Creation height < max input height (v3+)
    MonotonicHeightError,
    /// Negative creation height (v1+)
    NegativeHeight,
    /// Box exceeds 4KB limit
    BoxSizeExceeded,
    /// Script exceeds size limit
    ScriptSizeExceeded,
    /// Script verification failed
    ReducedToFalse,
    /// Verifier error
    VerifierError,
};
```

## Stateless Validation

Checks that don't require blockchain state[^3][^4]:

```zig
/// Validate transaction structure without blockchain state
pub fn validateStateless(tx: *const Transaction) TxValidationError!void {
    // BoundedVec ensures 1 ≤ count ≤ 32767, so no explicit checks needed

    // Check output sum doesn't overflow
    var output_sum: i64 = 0;
    for (tx.outputs.items()) |out| {
        output_sum = std.math.add(i64, output_sum, out.value.as_i64()) catch
            return error.OutputSumOverflow;
    }

    // Check no double-spend (unique inputs)
    var seen = std.AutoHashMap(BoxId, void).init(allocator);
    defer seen.deinit();

    for (tx.inputs.items()) |input| {
        const result = seen.getOrPut(input.box_id);
        if (result.found_existing) {
            return error.DoubleSpend;
        }
    }
}
```

### Stateless Rules Table

```
Stateless Validation Rules
─────────────────────────────────────────────────────
Rule              Check                    Limit
─────────────────────────────────────────────────────
txNoInputs        inputs.len >= 1          min 1
txNoOutputs       outputs.len >= 1         min 1
txManyInputs      inputs.len <= MAX        32,767
txManyDataInputs  data_inputs.len <= MAX   32,767
txManyOutputs     outputs.len <= MAX       32,767
txNegativeOutput  all outputs >= 0         -
txOutputSum       sum(outputs) no overflow -
txInputsUnique    no duplicate box_ids     -
─────────────────────────────────────────────────────
```

## Stateful Validation

Requires UTXO state and blockchain context[^5][^6]:

```zig
/// Validate transaction against blockchain state
pub fn validateStateful(
    tx: *const Transaction,
    boxes_to_spend: []const ErgoBox,
    data_boxes: []const ErgoBox,
    state_context: *const ErgoStateContext,
    accumulated_cost: u64,
    verifier: *const Verifier,
) TxValidationError!u64 {
    const params = state_context.current_parameters;
    const max_cost = params.max_block_cost;

    // 1. Calculate initial cost
    const initial_cost = calculateInitialCost(
        tx,
        boxes_to_spend.len,
        data_boxes.len,
        params,
    );
    var current_cost = accumulated_cost + initial_cost;

    if (current_cost > max_cost) {
        return error.CostExceeded;
    }

    // 2. Verify outputs
    const max_input_height = maxCreationHeight(boxes_to_spend);
    for (tx.outputs.items()) |out| {
        try verifyOutput(out, state_context, max_input_height);
    }

    // 3. Check ERG preservation
    const input_sum = try sumValues(boxes_to_spend);
    const output_sum = try sumValues(tx.outputs.items());
    if (input_sum != output_sum) {
        return error.ErgPreservationError;
    }

    // 4. Verify asset preservation
    current_cost = try verifyAssets(
        tx,
        boxes_to_spend,
        state_context,
        current_cost,
    );

    // 5. Verify each input script
    for (boxes_to_spend, 0..) |box, idx| {
        current_cost = try verifyInput(
            tx,
            boxes_to_spend,
            data_boxes,
            box,
            @intCast(idx),
            state_context,
            current_cost,
            verifier,
        );
    }

    return current_cost;
}
```

## Initial Cost Calculation

Transaction cost starts with fixed overhead[^7][^8]:

```zig
const CostConstants = struct {
    pub const INTERPRETER_INIT_COST: u64 = 10_000;
};

pub fn calculateInitialCost(
    tx: *const Transaction,
    inputs_count: usize,
    data_inputs_count: usize,
    params: *const BlockchainParameters,
) u64 {
    return CostConstants.INTERPRETER_INIT_COST +
        inputs_count * params.input_cost +
        data_inputs_count * params.data_input_cost +
        tx.outputs.len() * params.output_cost;
}
```

## Output Verification

Each output must pass structural checks[^9][^10]:

```zig
pub fn verifyOutput(
    out: *const ErgoBox,
    state_context: *const ErgoStateContext,
    max_input_height: u32,
) TxValidationError!void {
    const params = state_context.current_parameters;
    const block_version = state_context.block_version;
    const current_height = state_context.current_height;

    // Dust check: value >= minimum for box size
    const min_value = BoxUtils.minimalErgoAmount(out, params);
    if (out.value.as_u64() < min_value) {
        return error.DustOutput;
    }

    // Future check: creation height <= current height
    if (out.creation_height > current_height) {
        return error.InvalidHeightError;
    }

    // Non-negative height (after v1)
    if (block_version > 1 and out.creation_height < 0) {
        return error.NegativeHeight;
    }

    // Monotonic height (after v3): output height >= max input height
    if (block_version >= 3 and out.creation_height < max_input_height) {
        return error.MonotonicHeightError;
    }

    // Size limits
    if (out.serializedSize() > ErgoBox.MAX_BOX_SIZE) {
        return error.BoxSizeExceeded;
    }
    if (out.propositionBytes().len > ErgoBox.MAX_SCRIPT_SIZE) {
        return error.ScriptSizeExceeded;
    }
}
```

## Asset Verification

Token preservation rules[^11][^12]:

```zig
pub fn verifyAssets(
    tx: *const Transaction,
    boxes_to_spend: []const ErgoBox,
    state_context: *const ErgoStateContext,
    current_cost: u64,
) TxValidationError!u64 {
    // Extract input assets
    var in_assets = std.AutoHashMap(TokenId, u64).init(allocator);
    defer in_assets.deinit();

    for (boxes_to_spend) |box| {
        if (box.tokens) |tokens| {
            for (tokens.items()) |token| {
                const entry = in_assets.getOrPut(token.token_id);
                if (entry.found_existing) {
                    entry.value_ptr.* += token.amount.value;
                } else {
                    entry.value_ptr.* = token.amount.value;
                }
            }
        }
    }

    // Extract output assets
    var out_assets = std.AutoHashMap(TokenId, u64).init(allocator);
    defer out_assets.deinit();

    for (tx.outputs.items()) |out| {
        if (out.tokens) |tokens| {
            for (tokens.items()) |token| {
                const entry = out_assets.getOrPut(token.token_id);
                if (entry.found_existing) {
                    entry.value_ptr.* += token.amount.value;
                } else {
                    entry.value_ptr.* = token.amount.value;
                }
            }
        }
    }

    // First input box ID can mint new tokens
    const new_token_id = TokenId{ .digest = tx.inputs.items()[0].box_id.digest };

    // Verify each output token
    var iter = out_assets.iterator();
    while (iter.next()) |entry| {
        const out_id = entry.key_ptr.*;
        const out_amount = entry.value_ptr.*;

        const in_amount = in_assets.get(out_id) orelse 0;

        // Output amount <= input amount OR it's a new token
        if (out_amount > in_amount) {
            if (!std.mem.eql(u8, &out_id.digest, &new_token_id.digest) or out_amount == 0) {
                return error.TokenPreservationError;
            }
        }
    }

    // Add token access cost
    const token_access_cost = calculateTokenAccessCost(
        in_assets.count(),
        out_assets.count(),
        state_context.current_parameters.token_access_cost,
    );

    return current_cost + token_access_cost;
}
```

## Input Script Verification

The most expensive step—verify each input's script[^13][^14]:

```zig
pub fn verifyInput(
    tx: *const Transaction,
    boxes_to_spend: []const ErgoBox,
    data_boxes: []const ErgoBox,
    box: *const ErgoBox,
    input_index: u16,
    state_context: *const ErgoStateContext,
    current_cost: u64,
    verifier: *const Verifier,
) TxValidationError!u64 {
    const max_cost = state_context.current_parameters.max_block_cost;
    const input = tx.inputs.items()[input_index];
    const proof = input.spending_proof;

    // Check for storage rent spending first
    const ctx = try buildContext(
        tx,
        boxes_to_spend,
        data_boxes,
        input_index,
        state_context,
        max_cost - current_cost,
    );

    if (trySpendStorageRent(&input, box, state_context, &ctx)) |_| {
        // Storage rent conditions satisfied, skip script verification
        return current_cost + StorageConstants.STORAGE_CONTRACT_COST;
    }

    // Normal script verification
    const result = verifier.verify(
        &box.ergo_tree,
        &ctx,
        proof.proof,
        tx.messageToSign(),
    ) catch |err| {
        return error.VerifierError;
    };

    if (!result.result) {
        return error.ReducedToFalse;
    }

    const new_cost = current_cost + result.cost;
    if (new_cost > max_cost) {
        return error.CostExceeded;
    }

    return new_cost;
}
```

## Context Construction

Build evaluation context for input verification[^15][^16]:

```zig
pub fn buildContext(
    tx: *const Transaction,
    boxes_to_spend: []const ErgoBox,
    data_boxes: []const ErgoBox,
    input_index: u16,
    state_context: *const ErgoStateContext,
    cost_limit: u64,
) !Context {
    return Context{
        .height = state_context.pre_header.height,
        .self_box = &boxes_to_spend[input_index],
        .inputs = boxes_to_spend,
        .data_inputs = data_boxes,
        .outputs = tx.outputs.items(),
        .pre_header = &state_context.pre_header,
        .headers = state_context.headers,
        .extension = tx.contextExtension(input_index),
        .cost_limit = cost_limit,
        .tree_version = @intCast(state_context.block_version - 1),
    };
}
```

## Storage Rent Spending

Expired boxes can be spent without script verification[^17][^18]:

```zig
const StorageConstants = struct {
    /// Blocks before box is eligible (~4 years)
    pub const STORAGE_PERIOD: u32 = 1_051_200;
    /// Context extension key for output index
    pub const STORAGE_EXTENSION_INDEX: u8 = 127;
    /// Cost for storage rent verification
    pub const STORAGE_CONTRACT_COST: u64 = 50;
};

pub fn trySpendStorageRent(
    input: *const Input,
    input_box: *const ErgoBox,
    state_context: *const ErgoStateContext,
    ctx: *const Context,
) ?void {
    // Must have empty proof
    if (!input.spending_proof.proof.isEmpty()) return null;

    return checkStorageRentConditions(input_box, state_context, ctx);
}

pub fn checkStorageRentConditions(
    input_box: *const ErgoBox,
    state_context: *const ErgoStateContext,
    ctx: *const Context,
) ?void {
    // Check time elapsed
    const age = ctx.pre_header.height - ctx.self_box.creation_height;
    if (age < StorageConstants.STORAGE_PERIOD) return null;

    // Get output index from context extension
    const output_idx_value = ctx.extension.values.get(
        StorageConstants.STORAGE_EXTENSION_INDEX,
    ) orelse return null;
    const output_idx = output_idx_value.extractAs(i16) orelse return null;

    const output = ctx.outputs[@intCast(output_idx)];

    // Calculate storage fee
    const storage_fee = input_box.serializedSize() *
        state_context.parameters.storage_fee_factor;

    // Dust boxes can always be spent
    if (ctx.self_box.value.as_u64() <= storage_fee) return {};

    // Verify recreation rules
    if (output.creation_height != state_context.pre_header.height) return null;
    if (output.value.as_u64() < ctx.self_box.value.as_u64() - storage_fee) return null;

    // Registers must be preserved (except R0 value and R3 creation info)
    for (0..10) |i| {
        const reg_id = RegisterId.fromByte(@intCast(i));
        if (reg_id == .r0 or reg_id == .r3) continue;
        if (!std.meta.eql(
            ctx.self_box.getRegister(reg_id),
            output.getRegister(reg_id),
        )) return null;
    }

    return {};
}
```

## Cost Accumulation Flow

```
Cost Accumulation
─────────────────────────────────────────────────────

Block accumulated cost (from previous txs)
    │
    ├── + INTERPRETER_INIT_COST  (10,000)
    ├── + inputs.len × inputCost
    ├── + data_inputs.len × dataInputCost
    ├── + outputs.len × outputCost
    │
    ▼
startCost
    │
    ├── Input[0] script → + scriptCost₀
    ├── Input[1] script → + scriptCost₁
    ├── ...
    ├── Input[n] script → + scriptCostₙ
    │
    ├── Token access → + tokenAccessCost
    │
    ▼
finalCost ≤ maxBlockCost

Each input verification receives remaining budget:
  ctx.cost_limit = maxBlockCost - current_cost
```

## Validation Rules Summary

```
Validation Rules Reference
─────────────────────────────────────────────────────
ID    Name                    Phase       Description
─────────────────────────────────────────────────────
100   txNoInputs              Stateless   ≥1 input
101   txNoOutputs             Stateless   ≥1 output
102   txManyInputs            Stateless   ≤32,767
103   txManyDataInputs        Stateless   ≤32,767
104   txManyOutputs           Stateless   ≤32,767
105   txNegativeOutput        Stateless   values ≥ 0
106   txOutputSum             Stateless   no overflow
107   txInputsUnique          Stateless   no duplicates
─────────────────────────────────────────────────────
120   txScriptValidation      Stateful    scripts pass
121   bsBlockTransactionsCost Stateful    cost in limit
122   txDust                  Stateful    min value
123   txFuture                Stateful    valid height
124   txErgPreservation       Stateful    ERG balanced
125   txAssetsPreservation    Stateful    tokens balanced
126   txBoxSize               Stateful    ≤4KB
127   txReemission            Stateful    EIP-27 rules
─────────────────────────────────────────────────────
```

## Complete Validation Flow

```zig
/// Full transaction validation
pub fn validateTransaction(
    tx: *const Transaction,
    utxo_state: *const UtxoState,
    state_context: *const ErgoStateContext,
    verifier: *const Verifier,
    accumulated_cost: u64,
) !u64 {
    // Phase 1: Stateless validation
    try validateStateless(tx);

    // Phase 2: Resolve input boxes
    var boxes_to_spend: []ErgoBox = &.{};
    for (tx.inputs.items()) |input| {
        const box = utxo_state.boxById(input.box_id) orelse
            return error.InputBoxNotFound;
        boxes_to_spend = append(boxes_to_spend, box);
    }

    // Phase 3: Resolve data input boxes
    var data_boxes: []ErgoBox = &.{};
    if (tx.data_inputs) |data_inputs| {
        for (data_inputs.items()) |data_input| {
            const box = utxo_state.boxById(data_input.box_id) orelse
                return error.DataInputBoxNotFound;
            data_boxes = append(data_boxes, box);
        }
    }

    // Phase 4: Stateful validation
    return validateStateful(
        tx,
        boxes_to_spend,
        data_boxes,
        state_context,
        accumulated_cost,
        verifier,
    );
}
```

## Summary

- **Two-phase validation**: Stateless (structural) then stateful (UTXO-dependent)
- **Stateless**: Count limits, no negatives, no overflow, unique inputs
- **Stateful**: Cost tracking, output checks, preservation rules, script verification
- **Cost accumulation**: Tracks across inputs, bounded by maxBlockCost
- **Storage rent**: Expired boxes (~4 years) spendable by anyone with recreation
- **Asset preservation**: ERG exactly preserved, tokens can only decrease (or mint new)

---

*Next: [Chapter 25: Cost Limits and Parameters](./ch25-cost-limits-parameters.md)*

[^1]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:57-64`

[^2]: Rust: `ergo-lib/src/chain/transaction.rs:60-96`

[^3]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:91-115`

[^4]: Rust: `ergo-lib/src/chain/transaction/ergo_transaction.rs:93-110`

[^5]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:360-441`

[^6]: Rust: `ergo-lib/src/chain/transaction.rs:200-300`

[^7]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:370-374`

[^8]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoInterpreter.scala:93-96`

[^9]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:163-177`

[^10]: Rust: `ergo-lib/src/chain/transaction/ergo_transaction.rs:49-67`

[^11]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:180-216`

[^12]: Rust: `ergo-lib/src/chain/transaction/ergo_transaction.rs:37-48`

[^13]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:110-161`

[^14]: Rust: `ergo-lib/src/wallet/signing.rs:143-180`

[^15]: Scala: `ergo-core/src/main/scala/org/ergoplatform/nodeView/ErgoContext.scala:12-29`

[^16]: Rust: `ergo-lib/src/wallet/signing.rs:46-116`

[^17]: Scala: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoInterpreter.scala:42-55`

[^18]: Rust: `ergo-lib/src/chain/transaction/storage_rent.rs:12-78`
