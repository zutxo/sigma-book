# Chapter 25: Cost Limits and Parameters

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Transaction validation ([Chapter 24](./ch24-transaction-validation.md))
- Cost model ([Chapter 13](../part5/ch13-cost-model.md))

## Learning Objectives

- Understand Ergo's adjustable blockchain parameters
- Learn the voting mechanism for parameter changes
- Master cost-related parameters and defaults
- Understand validation rules configurability

## Parameter System

Ergo's blockchain parameters are adjustable through miner voting[^1][^2]:

```
Parameter Governance
─────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────┐
│                  Parameters                         │
├─────────────────────────────────────────────────────┤
│  parameters_table: HashMap<Parameter, i32>          │
│  proposed_update: ValidationSettingsUpdate          │
│  height: u32                                        │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│              Parameter Types                        │
├─────────────────────────────────────────────────────┤
│  Cost:     maxBlockCost, inputCost, outputCost...   │
│  Size:     maxBlockSize, minValuePerByte            │
│  Fee:      storageFeeFactor                         │
│  Version:  blockVersion                             │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│              Voting Mechanism                       │
├─────────────────────────────────────────────────────┤
│  Miners include votes in block headers              │
│  Votes tallied over epochs (1024 blocks)            │
│  Majority (>= 90%) activates change                 │
│  Each param has min/max bounds and step size        │
└─────────────────────────────────────────────────────┘
```

## Parameter Enum

```zig
const Parameter = enum(i8) {
    /// Storage fee factor (per byte per ~4 year storage period)
    storage_fee_factor = 1,
    /// Minimum monetary value per byte of box
    min_value_per_byte = 2,
    /// Maximum block size in bytes
    max_block_size = 3,
    /// Maximum computational cost per block
    max_block_cost = 4,
    /// Cost per token access
    token_access_cost = 5,
    /// Cost per transaction input
    input_cost = 6,
    /// Cost per data input
    data_input_cost = 7,
    /// Cost per transaction output
    output_cost = 8,
    /// Sub-blocks per block (v6+)
    subblocks_per_block = 9,
    /// Soft-fork vote
    soft_fork = 120,
    /// Soft-fork votes collected
    soft_fork_votes = 121,
    /// Soft-fork starting height
    soft_fork_start_height = 122,
    /// Current block version
    block_version = 123,

    /// Negative values indicate decrease vote
    pub fn decreaseVote(self: Parameter) i8 {
        return -@intFromEnum(self);
    }
};
```

## Parameters Structure

```zig
const Parameters = struct {
    /// Current block height
    height: u32,
    /// Parameter ID -> value mapping
    parameters_table: std.AutoHashMap(Parameter, i32),
    /// Proposed validation settings update
    proposed_update: ValidationSettingsUpdate,

    /// Get block version
    pub fn blockVersion(self: *const Parameters) i32 {
        return self.parameters_table.get(.block_version) orelse 1;
    }

    /// Get max block cost
    pub fn maxBlockCost(self: *const Parameters) i32 {
        return self.parameters_table.get(.max_block_cost) orelse DefaultParams.MAX_BLOCK_COST;
    }

    /// Get input cost
    pub fn inputCost(self: *const Parameters) i32 {
        return self.parameters_table.get(.input_cost) orelse DefaultParams.INPUT_COST;
    }

    /// Get data input cost
    pub fn dataInputCost(self: *const Parameters) i32 {
        return self.parameters_table.get(.data_input_cost) orelse DefaultParams.DATA_INPUT_COST;
    }

    /// Get output cost
    pub fn outputCost(self: *const Parameters) i32 {
        return self.parameters_table.get(.output_cost) orelse DefaultParams.OUTPUT_COST;
    }

    /// Get token access cost
    pub fn tokenAccessCost(self: *const Parameters) i32 {
        return self.parameters_table.get(.token_access_cost) orelse DefaultParams.TOKEN_ACCESS_COST;
    }

    /// Get storage fee factor
    pub fn storageFeeFactor(self: *const Parameters) i32 {
        return self.parameters_table.get(.storage_fee_factor) orelse DefaultParams.STORAGE_FEE_FACTOR;
    }

    /// Get min value per byte
    pub fn minValuePerByte(self: *const Parameters) i32 {
        return self.parameters_table.get(.min_value_per_byte) orelse DefaultParams.MIN_VALUE_PER_BYTE;
    }

    /// Get max block size
    pub fn maxBlockSize(self: *const Parameters) i32 {
        return self.parameters_table.get(.max_block_size) orelse DefaultParams.MAX_BLOCK_SIZE;
    }
};
```

## Default Values

```zig
const DefaultParams = struct {
    /// Cost parameters
    pub const MAX_BLOCK_COST: i32 = 1_000_000;
    pub const TOKEN_ACCESS_COST: i32 = 100;
    pub const INPUT_COST: i32 = 2_000;
    pub const DATA_INPUT_COST: i32 = 100;
    pub const OUTPUT_COST: i32 = 100;

    /// Size parameters
    pub const MAX_BLOCK_SIZE: i32 = 512 * 1024; // 512 KB
    pub const MAX_BLOCK_SIZE_MAX: i32 = 1024 * 1024; // 1 MB
    pub const MAX_BLOCK_SIZE_MIN: i32 = 16 * 1024; // 16 KB

    /// Fee parameters
    pub const STORAGE_FEE_FACTOR: i32 = 1_250_000; // 0.00125 ERG per byte per ~4 years
    pub const STORAGE_FEE_FACTOR_MAX: i32 = 2_500_000;
    pub const STORAGE_FEE_FACTOR_MIN: i32 = 0;
    pub const STORAGE_FEE_FACTOR_STEP: i32 = 25_000;

    /// Dust prevention
    pub const MIN_VALUE_PER_BYTE: i32 = 30 * 12; // 360 nanoErgs per byte
    pub const MIN_VALUE_PER_BYTE_MAX: i32 = 10_000;
    pub const MIN_VALUE_PER_BYTE_MIN: i32 = 0;
    pub const MIN_VALUE_PER_BYTE_STEP: i32 = 10;

    /// Sub-blocks (v6+)
    pub const SUBBLOCKS_PER_BLOCK: i32 = 30;
    pub const SUBBLOCKS_PER_BLOCK_MIN: i32 = 2;
    pub const SUBBLOCKS_PER_BLOCK_MAX: i32 = 2048;
    pub const SUBBLOCKS_PER_BLOCK_STEP: i32 = 1;

    /// Interpreter initialization cost
    pub const INTERPRETER_INIT_COST: i32 = 10_000;
};

/// Create default parameters
pub fn defaultParameters() Parameters {
    var table = std.AutoHashMap(Parameter, i32).init(allocator);
    table.put(.storage_fee_factor, DefaultParams.STORAGE_FEE_FACTOR) catch {};
    table.put(.min_value_per_byte, DefaultParams.MIN_VALUE_PER_BYTE) catch {};
    table.put(.token_access_cost, DefaultParams.TOKEN_ACCESS_COST) catch {};
    table.put(.input_cost, DefaultParams.INPUT_COST) catch {};
    table.put(.data_input_cost, DefaultParams.DATA_INPUT_COST) catch {};
    table.put(.output_cost, DefaultParams.OUTPUT_COST) catch {};
    table.put(.max_block_size, DefaultParams.MAX_BLOCK_SIZE) catch {};
    table.put(.max_block_cost, DefaultParams.MAX_BLOCK_COST) catch {};
    table.put(.block_version, 1) catch {};
    return Parameters{
        .height = 0,
        .parameters_table = table,
        .proposed_update = ValidationSettingsUpdate.empty(),
    };
}
```

## Parameter Reference

```
Default Parameter Values
─────────────────────────────────────────────────────
ID   Name                Default      Min        Max        Step
─────────────────────────────────────────────────────
1    storageFeeFactor    1,250,000    0          2,500,000  25,000
2    minValuePerByte     360          0          10,000     10
3    maxBlockSize        524,288      16,384     1,048,576  1%
4    maxBlockCost        1,000,000    16,384     -          1%
5    tokenAccessCost     100          -          -          1%
6    inputCost           2,000        -          -          1%
7    dataInputCost       100          -          -          1%
8    outputCost          100          -          -          1%
9    subblocksPerBlock   30           2          2,048      1
123  blockVersion        1            1          -          -
─────────────────────────────────────────────────────
```

## Voting Mechanism

Miners vote for parameter changes in block headers[^3]:

```zig
const VotingSettings = struct {
    /// Blocks per voting epoch
    pub const EPOCH_LENGTH: u32 = 1024;
    /// Required approval threshold (90%)
    pub const APPROVAL_THRESHOLD: f32 = 0.90;

    /// Check if vote count meets approval threshold
    pub fn changeApproved(self: *const VotingSettings, vote_count: u32) bool {
        const threshold = @as(u32, @intFromFloat(EPOCH_LENGTH * APPROVAL_THRESHOLD));
        return vote_count >= threshold;
    }
};

/// Generate votes based on targets
pub fn generateVotes(
    params: *const Parameters,
    own_targets: std.AutoHashMap(Parameter, i32),
    epoch_votes: []const struct { param: i8, count: u32 },
    vote_for_fork: bool,
) []i8 {
    var votes: []i8 = &.{};

    for (epoch_votes) |ev| {
        const param_id = ev.param;

        if (param_id == @intFromEnum(Parameter.soft_fork)) {
            if (vote_for_fork) {
                votes = append(votes, param_id);
            }
        } else if (param_id > 0) {
            // Vote for increase if current < target
            const param: Parameter = @enumFromInt(param_id);
            const current = params.parameters_table.get(param) orelse continue;
            const target = own_targets.get(param) orelse continue;
            if (target > current) {
                votes = append(votes, param_id);
            }
        } else if (param_id < 0) {
            // Vote for decrease if current > target
            const param: Parameter = @enumFromInt(-param_id);
            const current = params.parameters_table.get(param) orelse continue;
            const target = own_targets.get(param) orelse continue;
            if (target < current) {
                votes = append(votes, param_id);
            }
        }
    }

    return padVotes(votes);
}
```

## Parameter Update Logic

Apply votes at epoch boundaries[^4]:

```zig
/// Update parameters based on epoch votes
pub fn updateParams(
    params_table: std.AutoHashMap(Parameter, i32),
    epoch_votes: []const struct { param: i8, count: u32 },
    settings: *const VotingSettings,
) std.AutoHashMap(Parameter, i32) {
    var new_table = params_table.clone();

    for (epoch_votes) |ev| {
        const param_id = ev.param;
        if (param_id >= @intFromEnum(Parameter.soft_fork)) continue;

        const param_abs: Parameter = @enumFromInt(if (param_id < 0) -param_id else param_id);

        if (settings.changeApproved(ev.count)) {
            const current = new_table.get(param_abs) orelse continue;
            const max_val = getMaxValue(param_abs);
            const min_val = getMinValue(param_abs);
            const step = getStep(param_abs, current);

            const new_value = if (param_id > 0) blk: {
                // Increase: cap at max
                break :blk if (current < max_val) current + step else current;
            } else blk: {
                // Decrease: floor at min
                break :blk if (current > min_val) current - step else current;
            };

            new_table.put(param_abs, new_value) catch {};
        }
    }

    return new_table;
}

fn getMaxValue(param: Parameter) i32 {
    return switch (param) {
        .storage_fee_factor => DefaultParams.STORAGE_FEE_FACTOR_MAX,
        .min_value_per_byte => DefaultParams.MIN_VALUE_PER_BYTE_MAX,
        .max_block_size => DefaultParams.MAX_BLOCK_SIZE_MAX,
        .subblocks_per_block => DefaultParams.SUBBLOCKS_PER_BLOCK_MAX,
        else => std.math.maxInt(i32) / 2,
    };
}

fn getMinValue(param: Parameter) i32 {
    return switch (param) {
        .storage_fee_factor => DefaultParams.STORAGE_FEE_FACTOR_MIN,
        .min_value_per_byte => DefaultParams.MIN_VALUE_PER_BYTE_MIN,
        .max_block_size => DefaultParams.MAX_BLOCK_SIZE_MIN,
        .max_block_cost => 16 * 1024,
        .subblocks_per_block => DefaultParams.SUBBLOCKS_PER_BLOCK_MIN,
        else => 0,
    };
}

fn getStep(param: Parameter, current: i32) i32 {
    return switch (param) {
        .storage_fee_factor => DefaultParams.STORAGE_FEE_FACTOR_STEP,
        .min_value_per_byte => DefaultParams.MIN_VALUE_PER_BYTE_STEP,
        .subblocks_per_block => DefaultParams.SUBBLOCKS_PER_BLOCK_STEP,
        else => @max(1, @divTrunc(current, 100)), // Default 1% step
    };
}
```

## Cost Calculation

Transaction cost formula[^5][^6]:

```
Cost Formula
─────────────────────────────────────────────────────

totalCost = interpreterInitCost        // 10,000
          + inputs × inputCost         // inputs × 2,000
          + dataInputs × dataInputCost // dataInputs × 100
          + outputs × outputCost       // outputs × 100
          + tokenAccessCost × tokens   // varies
          + scriptExecutionCost        // varies per script

Example (2 inputs, 1 data input, 3 outputs, 50K script):
─────────────────────────────────────────────────────
  10,000  interpreter init
   4,000  2 × 2,000 inputs
     100  1 × 100 data inputs
     300  3 × 100 outputs
  50,000  script execution
─────────────────────────────────────────────────────
  64,400  TOTAL
```

```zig
/// Calculate transaction cost
pub fn calculateTransactionCost(
    params: *const Parameters,
    num_inputs: usize,
    num_data_inputs: usize,
    num_outputs: usize,
    script_cost: u64,
    token_ops: usize,
) u64 {
    const init_cost = DefaultParams.INTERPRETER_INIT_COST;
    const input_cost = params.inputCost() * @as(i32, @intCast(num_inputs));
    const data_input_cost = params.dataInputCost() * @as(i32, @intCast(num_data_inputs));
    const output_cost = params.outputCost() * @as(i32, @intCast(num_outputs));
    const token_cost = params.tokenAccessCost() * @as(i32, @intCast(token_ops));

    return @intCast(init_cost + input_cost + data_input_cost + output_cost + token_cost) + script_cost;
}

/// Calculate block capacity in simple transactions
pub fn estimateBlockCapacity(params: *const Parameters) u32 {
    const max_cost = params.maxBlockCost();

    // Simple tx: 1 input (P2PK), 2 outputs, ~15K script cost
    const simple_tx_cost = DefaultParams.INTERPRETER_INIT_COST +
        params.inputCost() +
        params.outputCost() * 2 +
        15_000; // P2PK verification

    return @intCast(@divTrunc(max_cost, simple_tx_cost));
}
```

## Block Version History

```
Protocol Versions
─────────────────────────────────────────────────────
Block Version   Protocol   Features
─────────────────────────────────────────────────────
1               v1         Initial mainnet
2               v5.0       Script improvements
3               v5.0.12    Height monotonicity (EIP-39)
4               v6.0       Sub-blocks, new operations
─────────────────────────────────────────────────────

Script version = block_version - 1
```

## Validation Rules

Rules can be enabled/disabled via soft-fork[^7]:

```zig
const RuleStatus = struct {
    /// Creates error from modifier details
    create_error: fn (InvalidModifier) Invalid,
    /// Which modifier types this rule applies to
    affected_classes: []const ModifierType,
    /// Can this rule be disabled via soft-fork?
    may_be_disabled: bool,
    /// Is this rule currently active?
    is_active: bool,
};

/// Validation rule IDs
const ValidationRules = struct {
    // Stateless (100-109)
    pub const TX_NO_INPUTS: u16 = 100;
    pub const TX_NO_OUTPUTS: u16 = 101;
    pub const TX_MANY_INPUTS: u16 = 102;
    pub const TX_MANY_DATA_INPUTS: u16 = 103;
    pub const TX_MANY_OUTPUTS: u16 = 104;
    pub const TX_NEGATIVE_OUTPUT: u16 = 105;
    pub const TX_OUTPUT_SUM: u16 = 106;
    pub const TX_INPUTS_UNIQUE: u16 = 107;
    pub const TX_POSITIVE_ASSETS: u16 = 108;
    pub const TX_ASSETS_IN_ONE_BOX: u16 = 109;

    // Stateful (111-127)
    pub const TX_DUST: u16 = 111;
    pub const TX_FUTURE: u16 = 112;
    pub const TX_BOXES_TO_SPEND: u16 = 113;
    pub const TX_DATA_BOXES: u16 = 114;
    pub const TX_INPUTS_SUM: u16 = 115;
    pub const TX_ERG_PRESERVATION: u16 = 116;
    pub const TX_ASSETS_PRESERVATION: u16 = 117;
    pub const TX_BOX_TO_SPEND: u16 = 118;
    pub const TX_SCRIPT_VALIDATION: u16 = 119;
    pub const TX_BOX_SIZE: u16 = 120;
    pub const TX_BOX_PROPOSITION_SIZE: u16 = 121;
    pub const TX_NEG_HEIGHT: u16 = 122; // v2+
    pub const TX_REEMISSION: u16 = 123; // EIP-27
    pub const TX_MONOTONIC_HEIGHT: u16 = 124; // v3+

    // Block rules (300+)
    pub const BS_BLOCK_TX_SIZE: u16 = 306;
    pub const BS_BLOCK_TX_COST: u16 = 307;
};
```

## Rule Configurability

```
Rule Categories
─────────────────────────────────────────────────────
Category              Can Disable?  Examples
─────────────────────────────────────────────────────
Consensus Critical    No            txErgPreservation
                                    txScriptValidation
                                    txNoInputs

Soft-Forkable         Yes           txDust
                                    txBoxSize
                                    txReemission

Version-Gated         N/A           txNegHeight (v2+)
                                    txMonotonicHeight (v3+)
─────────────────────────────────────────────────────
```

```zig
/// Check if rule can be disabled
pub fn mayBeDisabled(rule: u16) bool {
    return switch (rule) {
        ValidationRules.TX_DUST,
        ValidationRules.TX_BOX_SIZE,
        ValidationRules.TX_BOX_PROPOSITION_SIZE,
        ValidationRules.TX_REEMISSION,
        => true,

        // Consensus-critical rules cannot be disabled
        ValidationRules.TX_NO_INPUTS,
        ValidationRules.TX_ERG_PRESERVATION,
        ValidationRules.TX_SCRIPT_VALIDATION,
        ValidationRules.TX_ASSETS_PRESERVATION,
        => false,

        else => false,
    };
}
```

## Parameter Serialization

Parameters stored in block extensions[^8]:

```zig
const SYSTEM_PARAMETERS_PREFIX: u8 = 0x00;
const SOFT_FORK_DISABLING_RULES_KEY: [2]u8 = .{ 0x00, 0x01 };

/// Serialize parameters to extension candidate
pub fn toExtensionCandidate(params: *const Parameters) ExtensionCandidate {
    var fields: []ExtensionField = &.{};

    // Add parameter fields
    var iter = params.parameters_table.iterator();
    while (iter.next()) |entry| {
        const key = [2]u8{ SYSTEM_PARAMETERS_PREFIX, @intFromEnum(entry.key_ptr.*) };
        const value = std.mem.toBytes(@byteSwap(entry.value_ptr.*));
        fields = append(fields, ExtensionField{ .key = key, .value = &value });
    }

    // Add proposed update
    const update_bytes = params.proposed_update.serialize();
    fields = append(fields, ExtensionField{
        .key = SOFT_FORK_DISABLING_RULES_KEY,
        .value = update_bytes,
    });

    return ExtensionCandidate{ .fields = fields };
}

/// Parse parameters from extension
pub fn parseExtension(height: u32, extension: *const Extension) !Parameters {
    var params_table = std.AutoHashMap(Parameter, i32).init(allocator);

    for (extension.fields) |field| {
        if (field.key[0] == SYSTEM_PARAMETERS_PREFIX and
            field.key[1] != SOFT_FORK_DISABLING_RULES_KEY[1])
        {
            const param: Parameter = @enumFromInt(field.key[1]);
            const value = @byteSwap(std.mem.bytesToValue(i32, field.value[0..4]));
            try params_table.put(param, value);
        }
    }

    var proposed_update = ValidationSettingsUpdate.empty();
    for (extension.fields) |field| {
        if (std.mem.eql(u8, &field.key, &SOFT_FORK_DISABLING_RULES_KEY)) {
            proposed_update = try ValidationSettingsUpdate.parse(field.value);
            break;
        }
    }

    return Parameters{
        .height = height,
        .parameters_table = params_table,
        .proposed_update = proposed_update,
    };
}
```

## Summary

- **Parameters adjustable** via miner voting (1024-block epochs, 90% threshold)
- **Cost parameters**: maxBlockCost (1M), inputCost (2K), outputCost (100)
- **Size parameters**: maxBlockSize (512KB), minValuePerByte (360)
- **Fee parameters**: storageFeeFactor (1.25M nanoErgs per byte per ~4 years)
- **Block version** tracks protocol upgrades (script_version = block_version - 1)
- **Validation rules** can be consensus-critical or soft-forkable
- Parameters stored in **block extensions**, parsed at epoch boundaries

---

*Next: [Chapter 26: Wallet and Signing](./ch26-wallet-signing.md)*

[^1]: Scala: `ergo-core/src/main/scala/org/ergoplatform/settings/Parameters.scala:23-26`

[^2]: Rust: `ergo-lib/src/chain/parameters.rs:8-27`

[^3]: Scala: `ergo-core/src/main/scala/org/ergoplatform/settings/Parameters.scala:190-217`

[^4]: Scala: `ergo-core/src/main/scala/org/ergoplatform/settings/Parameters.scala:159-183`

[^5]: Scala: `ergo-core/src/main/scala/org/ergoplatform/modifiers/mempool/ErgoTransaction.scala:370-374`

[^6]: Rust: `ergo-lib/src/chain/parameters.rs:62-77`

[^7]: Scala: `ergo-core/src/main/scala/org/ergoplatform/settings/ValidationRules.scala:234-327`

[^8]: Scala: `ergo-core/src/main/scala/org/ergoplatform/settings/Parameters.scala:220-228`