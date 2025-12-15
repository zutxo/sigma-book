# Appendix C: Cost Table

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


Complete reference for operation costs in the JIT cost model[^1][^2].

## Cost Model Architecture

```
Cost Model Structure
══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│                       CostKind                                  │
├─────────────┬─────────────┬─────────────┬─────────────────────┤
│  FixedCost  │PerItemCost  │TypeBasedCost│   DynamicCost       │
│             │             │             │                      │
│  cost: u32  │ base: u32   │ costFunc()  │ sum of sub-costs    │
│             │ per_chunk   │ per type    │                      │
│             │ chunk_size  │             │                      │
└─────────────┴─────────────┴─────────────┴─────────────────────┘

Cost Calculation Flow:
─────────────────────────────────────────────────────────────────
                 ┌─────────────┐
                 │  Operation  │
                 └──────┬──────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
    ┌─────────┐   ┌──────────┐   ┌─────────┐
    │ FixedOp │   │PerItemOp │   │TypedOp  │
    │ cost=26 │   │base=20   │   │depends  │
    └────┬────┘   │chunk=10  │   │on type  │
         │        └────┬─────┘   └────┬────┘
         │             │              │
         └─────────────┼──────────────┘
                       ▼
              ┌────────────────┐
              │CostAccumulator │
              │ accum += cost  │
              │ check < limit  │
              └────────────────┘
```

## Zig Cost Types

```zig
const JitCost = struct {
    value: u32,

    pub fn add(self: JitCost, other: JitCost) !JitCost {
        return .{ .value = try std.math.add(u32, self.value, other.value) };
    }
};

const CostKind = union(enum) {
    fixed: FixedCost,
    per_item: PerItemCost,
    type_based: TypeBasedCost,
    dynamic,

    pub fn compute(self: CostKind, ctx: CostContext) JitCost {
        return switch (self) {
            .fixed => |f| f.cost,
            .per_item => |p| p.compute(ctx.n_items),
            .type_based => |t| t.costFunc(ctx.tpe),
            .dynamic => ctx.computed_cost,
        };
    }
};

/// Fixed cost regardless of input
const FixedCost = struct {
    cost: JitCost,
};

/// Cost proportional to collection size
const PerItemCost = struct {
    base: JitCost,
    per_chunk: JitCost,
    chunk_size: usize,

    /// totalCost = base + per_chunk * ceil(n_items / chunk_size)
    pub fn compute(self: PerItemCost, n_items: usize) JitCost {
        const chunks = (n_items + self.chunk_size - 1) / self.chunk_size;
        return .{
            .value = self.base.value + @as(u32, @intCast(chunks)) * self.per_chunk.value,
        };
    }
};

/// Cost depends on type
const TypeBasedCost = struct {
    primitive_cost: JitCost,
    bigint_cost: JitCost,
    collection_cost: ?PerItemCost,

    pub fn costFunc(self: TypeBasedCost, tpe: SType) JitCost {
        return switch (tpe) {
            .byte, .short, .int, .long => self.primitive_cost,
            .big_int, .unsigned_big_int => self.bigint_cost,
            .coll => |elem| if (self.collection_cost) |c|
                c.compute(elem.len)
            else
                self.primitive_cost,
            else => self.primitive_cost,
        };
    }
};
```

## Cost Accumulator

```zig
const CostAccumulator = struct {
    accum: u64,
    limit: u64,

    pub fn init(limit: u64) CostAccumulator {
        return .{ .accum = 0, .limit = limit };
    }

    pub fn add(self: *CostAccumulator, cost: JitCost) !void {
        self.accum += cost.value;
        if (self.accum > self.limit) {
            return error.CostLimitExceeded;
        }
    }

    pub fn addSeq(
        self: *CostAccumulator,
        cost: PerItemCost,
        n_items: usize,
    ) !void {
        try self.add(cost.compute(n_items));
    }

    pub fn totalCost(self: *const CostAccumulator) JitCost {
        return .{ .value = @intCast(self.accum) };
    }
};
```

## Fixed Cost Operations

| Operation | Cost | Description |
|-----------|------|-------------|
| ConstantPlaceholder | 1 | Reference segregated constant |
| Height | 1 | Current block height |
| Inputs | 1 | Transaction inputs |
| Outputs | 1 | Transaction outputs |
| LastBlockUtxoRootHash | 1 | UTXO root hash |
| Self | 1 | Self box |
| MinerPubkey | 1 | Miner public key |
| ValUse | 5 | Use defined value |
| TaggedVariable | 5 | Context variable |
| SomeValue | 5 | Option Some |
| NoneValue | 5 | Option None |
| SelectField | 8 | Select tuple field |
| CreateProveDlog | 10 | Create DLog |
| OptionGetOrElse | 10 | Option.getOrElse |
| OptionIsDefined | 10 | Option.isDefined |
| OptionGet | 10 | Option.get |
| ExtractAmount | 10 | Box value |
| ExtractScriptBytes | 10 | Proposition bytes |
| ExtractId | 10 | Box ID |
| Tuple | 10 | Create tuple |
| Select1-5 | 12 | Select tuple element |
| ByIndex | 14 | Collection access |
| BoolToSigmaProp | 15 | Bool → SigmaProp |
| DeserializeContext | 15 | Deserialize context |
| DeserializeRegister | 15 | Deserialize register |
| ByteArrayToLong | 16 | Bytes → Long |
| LongToByteArray | 17 | Long → bytes |
| CreateProveDHTuple | 20 | Create DHT |
| If | 20 | Conditional |
| LogicalNot | 20 | Boolean NOT |
| Negation | 20 | Numeric negation |
| ArithOp | 26 | Plus, Minus, etc. |
| ByteArrayToBigInt | 30 | Bytes → BigInt |
| SubstConstants | 30 | Substitute constants |
| SizeOf | 30 | Collection size |
| MultiplyGroup | 40 | EC point multiply |
| ExtractRegisterAs | 50 | Register access |
| Exponentiate | 300 | BigInt exponent |
| DecodePoint | 900 | Decode EC point |

## Per-Item Cost Operations

| Operation | Base | Per Chunk | Chunk Size |
|-----------|------|-----------|------------|
| CalcBlake2b256 | 20 | 7 | 128 |
| CalcSha256 | 20 | 8 | 64 |
| MapCollection | 20 | 1 | 10 |
| Exists | 20 | 5 | 10 |
| ForAll | 20 | 5 | 10 |
| Fold | 20 | 1 | 10 |
| Filter | 20 | 5 | 10 |
| FlatMap | 20 | 5 | 10 |
| Slice | 10 | 2 | 100 |
| Append | 20 | 2 | 100 |
| SigmaAnd | 10 | 2 | 1 |
| SigmaOr | 10 | 2 | 1 |
| AND (logical) | 10 | 5 | 32 |
| OR (logical) | 10 | 5 | 32 |
| XorOf | 20 | 5 | 32 |
| AtLeast | 20 | 3 | 1 |
| Xor (bytes) | 10 | 2 | 128 |

## Type-Based Costs

### Numeric Casting

| Target Type | Cost |
|-------------|------|
| Byte, Short, Int, Long | 10 |
| BigInt | 30 |
| UnsignedBigInt | 30 |

### Comparison Operations

| Type | Cost |
|------|------|
| Primitives | 10-20 |
| BigInt | 30 |
| Collections | PerItemCost |
| Tuples | Sum of components |

## Interpreter Overhead

| Cost Type | Value | Description |
|-----------|-------|-------------|
| interpreterInitCost | 10,000 | Interpreter init |
| inputCost | 2,000 | Per input |
| dataInputCost | 100 | Per data input |
| outputCost | 100 | Per output |
| tokenAccessCost | 100 | Per token |

## Cost Limits

| Parameter | Value | Description |
|-----------|-------|-------------|
| maxBlockCost | 1,000,000 | Max per block |
| scriptCostLimit | ~8,000,000 | Single script |

## Zig Cost Constants

```zig
const OperationCosts = struct {
    // Context access (very cheap)
    pub const HEIGHT: FixedCost = .{ .cost = .{ .value = 1 } };
    pub const INPUTS: FixedCost = .{ .cost = .{ .value = 1 } };
    pub const OUTPUTS: FixedCost = .{ .cost = .{ .value = 1 } };
    pub const SELF: FixedCost = .{ .cost = .{ .value = 1 } };

    // Variable access
    pub const VAL_USE: FixedCost = .{ .cost = .{ .value = 5 } };
    pub const CONSTANT_PLACEHOLDER: FixedCost = .{ .cost = .{ .value = 1 } };

    // Arithmetic
    pub const ARITH_OP: FixedCost = .{ .cost = .{ .value = 26 } };
    pub const COMPARISON: FixedCost = .{ .cost = .{ .value = 20 } };

    // Box extraction
    pub const EXTRACT_AMOUNT: FixedCost = .{ .cost = .{ .value = 10 } };
    pub const EXTRACT_REGISTER: FixedCost = .{ .cost = .{ .value = 50 } };

    // Cryptographic
    pub const PROVE_DLOG: FixedCost = .{ .cost = .{ .value = 10 } };
    pub const PROVE_DHT: FixedCost = .{ .cost = .{ .value = 20 } };
    pub const DECODE_POINT: FixedCost = .{ .cost = .{ .value = 900 } };
    pub const MULTIPLY_GROUP: FixedCost = .{ .cost = .{ .value = 40 } };
    pub const EXPONENTIATE: FixedCost = .{ .cost = .{ .value = 300 } };

    // Hashing
    pub const BLAKE2B256: PerItemCost = .{
        .base = .{ .value = 20 },
        .per_chunk = .{ .value = 7 },
        .chunk_size = 128,
    };
    pub const SHA256: PerItemCost = .{
        .base = .{ .value = 20 },
        .per_chunk = .{ .value = 8 },
        .chunk_size = 64,
    };

    // Collection operations
    pub const MAP: PerItemCost = .{
        .base = .{ .value = 20 },
        .per_chunk = .{ .value = 1 },
        .chunk_size = 10,
    };
    pub const FILTER: PerItemCost = .{
        .base = .{ .value = 20 },
        .per_chunk = .{ .value = 5 },
        .chunk_size = 10,
    };
    pub const FOLD: PerItemCost = .{
        .base = .{ .value = 20 },
        .per_chunk = .{ .value = 1 },
        .chunk_size = 10,
    };

    // Sigma operations
    pub const SIGMA_AND: PerItemCost = .{
        .base = .{ .value = 10 },
        .per_chunk = .{ .value = 2 },
        .chunk_size = 1,
    };
    pub const SIGMA_OR: PerItemCost = .{
        .base = .{ .value = 10 },
        .per_chunk = .{ .value = 2 },
        .chunk_size = 1,
    };
};

const InterpreterCosts = struct {
    pub const INIT: u32 = 10_000;
    pub const PER_INPUT: u32 = 2_000;
    pub const PER_DATA_INPUT: u32 = 100;
    pub const PER_OUTPUT: u32 = 100;
    pub const PER_TOKEN: u32 = 100;
};

const CostLimits = struct {
    pub const MAX_BLOCK_COST: u64 = 1_000_000;
    pub const MAX_SCRIPT_COST: u64 = 8_000_000;
};
```

## Cost Calculation Example

```zig
/// Calculate total cost for transaction verification
fn calculateTxCost(
    n_inputs: usize,
    n_data_inputs: usize,
    n_outputs: usize,
    script_costs: []const JitCost,
) u64 {
    var total: u64 = InterpreterCosts.INIT;

    total += @as(u64, n_inputs) * InterpreterCosts.PER_INPUT;
    total += @as(u64, n_data_inputs) * InterpreterCosts.PER_DATA_INPUT;
    total += @as(u64, n_outputs) * InterpreterCosts.PER_OUTPUT;

    for (script_costs) |cost| {
        total += cost.value;
    }

    return total;
}

// Example: 2 inputs, 1 data input, 3 outputs
// Base: 10,000 + 4,000 + 100 + 300 = 14,400
// Plus script costs per input
```

---

*[Previous: Appendix B](./appendix-b-opcodes.md) | [Next: Appendix D](./appendix-d-methods.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/ast/CostKind.scala`

[^2]: Rust: `ergotree-interpreter/src/eval/cost_accum.rs:7-43`