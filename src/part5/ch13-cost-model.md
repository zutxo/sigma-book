# Chapter 13: Cost Model

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 12](./ch12-evaluation-model.md) for the evaluation architecture and how costs are accumulated during eval
- [Chapter 5](../part2/ch05-operations-opcodes.md) for operation categories and cost descriptor types
- Basic computational complexity: understanding of constant-time vs linear-time operations

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain JitCost scaling (10x) and conversion to/from block costs
- Apply the three cost descriptor types: `FixedCost`, `PerItemCost`, and `TypeBasedCost`
- Implement cost accumulation with limit enforcement to prevent denial-of-service attacks
- Use cost tracing to analyze script execution costs

## Cost Model Purpose

Unlike Turing-complete smart contract platforms that can enter infinite loops, ErgoTree scripts must terminate within bounded resources. The cost model assigns a computational cost to every operation, accumulating these costs during evaluation. If the accumulated cost exceeds the block limit, execution fails—this guarantees that all scripts terminate and prevents attackers from crafting expensive scripts that slow down block validation.

ErgoTree scripts execute in a resource-constrained environment[^1][^2]:

```
Cost Model Guarantees
─────────────────────────────────────────────────────
1. DoS Protection     Expensive scripts blocked
2. Predictable Time   Miners estimate validation
3. Fair Pricing       Users pay for resources
4. Bounded Verify     All scripts terminate
```

## JitCost: The Cost Unit

JitCost provides 10x finer granularity than block costs[^3]:

```zig
const JitCost = struct {
    value: i32,

    pub const SCALE_FACTOR: i32 = 10;

    /// Add with overflow protection
    pub fn add(self: JitCost, other: JitCost) !JitCost {
        const result = @addWithOverflow(self.value, other.value);
        if (result[1] != 0) return error.CostOverflow;
        return .{ .value = result[0] };
    }

    /// Multiply with overflow protection
    pub fn mul(self: JitCost, n: i32) !JitCost {
        const result = @mulWithOverflow(self.value, n);
        if (result[1] != 0) return error.CostOverflow;
        return .{ .value = result[0] };
    }

    /// Divide by integer
    pub fn div(self: JitCost, n: i32) JitCost {
        return .{ .value = @divTrunc(self.value, n) };
    }

    /// Convert to block cost (inverse of fromBlockCost)
    pub fn toBlockCost(self: JitCost) i32 {
        return @divTrunc(self.value, SCALE_FACTOR);
    }

    /// Create from block cost
    pub fn fromBlockCost(block_cost: i32) !JitCost {
        const result = @mulWithOverflow(block_cost, SCALE_FACTOR);
        if (result[1] != 0) return error.CostOverflow;
        return .{ .value = result[0] };
    }

    /// Comparison
    pub fn gt(self: JitCost, other: JitCost) bool {
        return self.value > other.value;
    }
};
```

### Cost Scaling

```
Cost Scales
─────────────────────────────────────────────────────

JitCost (internal)    ─────────────────>    Block Cost
                          ÷ 10

Example:
  JitCost(50)   ──────────────────────>   5 block units
  JitCost(123)  ──────────────────────>   12 block units

Block Cost (external) <─────────────────    JitCost
                          × 10
```

The 10x scaling provides:
- **Finer granularity** for internal calculations
- **Integer arithmetic** (no floating point)
- **Overflow protection** via checked operations

## Cost Kind Descriptors

Cost descriptors define how operations are costed[^4][^5]:

```zig
const CostKind = union(enum) {
    fixed: FixedCost,
    per_item: PerItemCost,
    type_based: TypeBasedCost,
    dynamic: void,
};

/// Constant time operations
const FixedCost = struct {
    cost: JitCost,
};

/// Linear operations with chunking
const PerItemCost = struct {
    base_cost: JitCost,
    per_chunk_cost: JitCost,
    chunk_size: u32,

    /// Compute number of chunks for n items
    pub fn chunks(self: PerItemCost, n_items: usize) usize {
        if (n_items == 0) return 1;
        return (n_items - 1) / self.chunk_size + 1;
    }

    /// Compute total cost for n items
    pub fn cost(self: PerItemCost, n_items: usize) !JitCost {
        const n_chunks = self.chunks(n_items);
        const chunk_cost = try self.per_chunk_cost.mul(@intCast(n_chunks));
        return self.base_cost.add(chunk_cost);
    }
};

/// Type-dependent operations
const TypeBasedCost = struct {
    cost_fn: *const fn (SType) JitCost,
};
```

### FixedCost Operations

| Operation | Cost | Description |
|-----------|------|-------------|
| Constant | 5 | Return constant value |
| ConstantPlaceholder | 1 | Lookup segregated constant |
| ValUse | 5 | Variable lookup |
| If | 10 | Conditional branch |
| SelectField | 10 | Tuple field access |
| SizeOf | 14 | Get collection size |

### PerItemCost Operations

| Operation | Base | Per Chunk | Chunk Size |
|-----------|------|-----------|------------|
| blake2b256 | 20 | 7 | 128 bytes |
| sha256 | 80 | 8 | 64 bytes |
| Append | 20 | 2 | 10 items |
| Filter | 20 | 2 | 10 items |
| Map | 20 | 2 | 10 items |
| Fold | 20 | 2 | 10 items |

### Cost Formula

```
PerItemCost Formula
─────────────────────────────────────────────────────

total = baseCost + ceil(nItems / chunkSize) × perChunkCost

Example: Map over 50 elements
  chunks = ceil(50 / 10) = 5
  cost = 20 + 5 × 2 = 30 JitCost units
```

## Type-Based Costs

Operations with type-dependent complexity[^6]:

```zig
/// Numeric cast cost depends on target type
const NumericCastCost = struct {
    pub fn costFunc(target_type: SType) JitCost {
        return switch (target_type) {
            .s_big_int, .s_unsigned_big_int => .{ .value = 30 },
            else => .{ .value = 10 }, // Byte, Short, Int, Long
        };
    }
};

/// Equality cost depends on operand types
const EqualityCost = struct {
    pub fn costFunc(tpe: SType) JitCost {
        return switch (tpe) {
            .s_byte, .s_short, .s_int, .s_long => .{ .value = 3 },
            .s_big_int => .{ .value = 6 },
            .s_group_element => .{ .value = 172 },
            .s_coll => |elem| blk: {
                // Recursive: base + per-element
                const elem_cost = costFunc(elem.*);
                break :blk .{ .value = 10 + elem_cost.value };
            },
            else => .{ .value = 10 },
        };
    }
};
```

## Cost Items: Tracing

Cost items record individual contributions for debugging[^7][^8]:

```zig
const CostItem = union(enum) {
    fixed: FixedCostItem,
    seq: SeqCostItem,
    type_based: TypeBasedCostItem,

    pub fn opName(self: CostItem) []const u8 {
        return switch (self) {
            .fixed => |f| f.op_desc.name,
            .seq => |s| s.op_desc.name,
            .type_based => |t| t.op_desc.name,
        };
    }

    pub fn cost(self: CostItem) JitCost {
        return switch (self) {
            .fixed => |f| f.cost_kind.cost,
            .seq => |s| s.cost_kind.cost(s.n_items) catch .{ .value = 0 },
            .type_based => |t| t.cost_kind.cost_fn(t.tpe),
        };
    }
};

const FixedCostItem = struct {
    op_desc: OperationDesc,
    cost_kind: FixedCost,
};

const SeqCostItem = struct {
    op_desc: OperationDesc,
    cost_kind: PerItemCost,
    n_items: usize,

    pub fn chunks(self: SeqCostItem) usize {
        return self.cost_kind.chunks(self.n_items);
    }
};

const TypeBasedCostItem = struct {
    op_desc: OperationDesc,
    cost_kind: TypeBasedCost,
    tpe: SType,
};
```

## Cost Accumulator

Tracks costs during evaluation with limit enforcement[^9][^10]:

```zig
const CostCounter = struct {
    initial_cost: JitCost,
    current_cost: JitCost,

    pub fn init(initial: JitCost) CostCounter {
        return .{
            .initial_cost = initial,
            .current_cost = initial,
        };
    }

    pub fn add(self: *CostCounter, cost: JitCost) !void {
        self.current_cost = try self.current_cost.add(cost);
    }

    pub fn reset(self: *CostCounter) void {
        self.current_cost = self.initial_cost;
    }
};

const CostAccumulator = struct {
    scope_stack: std.ArrayList(Scope),
    cost_limit: ?JitCost,
    allocator: Allocator,

    const Scope = struct {
        counter: CostCounter,
        child_result: i32 = 0,

        pub fn add(self: *Scope, cost: JitCost) !void {
            try self.counter.add(cost);
        }

        pub fn currentCost(self: *const Scope) JitCost {
            return self.counter.current_cost;
        }
    };

    pub fn init(
        allocator: Allocator,
        initial_cost: JitCost,
        cost_limit: ?JitCost,
    ) CostAccumulator {
        var stack = std.ArrayList(Scope).init(allocator);
        stack.append(.{ .counter = CostCounter.init(initial_cost) }) catch unreachable;
        return .{
            .scope_stack = stack,
            .cost_limit = cost_limit,
            .allocator = allocator,
        };
    }

    pub fn currentScope(self: *CostAccumulator) *Scope {
        return &self.scope_stack.items[self.scope_stack.items.len - 1];
    }

    /// Add cost, checking limit
    pub fn add(self: *CostAccumulator, cost: JitCost) !void {
        try self.currentScope().add(cost);

        if (self.cost_limit) |limit| {
            const accumulated = self.currentScope().currentCost();
            if (accumulated.gt(limit)) {
                return error.CostLimitExceeded;
            }
        }
    }

    /// Total accumulated cost
    pub fn totalCost(self: *const CostAccumulator) JitCost {
        return self.scope_stack.items[self.scope_stack.items.len - 1].counter.current_cost;
    }

    pub fn reset(self: *CostAccumulator) void {
        self.scope_stack.clearRetainingCapacity();
        self.scope_stack.append(.{
            .counter = CostCounter.init(.{ .value = 0 }),
        }) catch unreachable;
    }
};
```

### Cost Limit Enforcement

```
Cost Accumulation Flow
─────────────────────────────────────────────────────

Each operation:
  1. Compute operation cost
  2. Call accumulator.add(opCost)
  3. Check: accumulatedCost > limit?
     Yes → return CostLimitExceeded
     No  → continue execution

At the end:
  totalCost = accumulator.totalCost()
  blockCost = totalCost.toBlockCost()
```

## Evaluator Cost Methods

The evaluator provides methods to add costs[^11][^12]:

```zig
const Evaluator = struct {
    cost_accum: CostAccumulator,
    cost_trace: ?std.ArrayList(CostItem),
    profiler: ?*Profiler,

    // ... other fields

    /// Add fixed cost
    pub fn addCost(self: *Evaluator, cost_kind: FixedCost, op_desc: OperationDesc) !void {
        try self.cost_accum.add(cost_kind.cost);

        if (self.cost_trace) |*trace| {
            try trace.append(.{
                .fixed = .{ .op_desc = op_desc, .cost_kind = cost_kind },
            });
        }
    }

    /// Add fixed cost and execute block
    pub fn addFixedCost(
        self: *Evaluator,
        cost_kind: FixedCost,
        op_desc: OperationDesc,
        comptime block: fn (*Evaluator) anyerror!anytype,
    ) !@TypeOf(block(self)) {
        if (self.profiler) |prof| {
            const start = std.time.nanoTimestamp();
            try self.cost_accum.add(cost_kind.cost);
            const result = try block(self);
            const end = std.time.nanoTimestamp();
            prof.addTiming(op_desc, end - start);
            return result;
        } else {
            try self.cost_accum.add(cost_kind.cost);
            return block(self);
        }
    }

    /// Add per-item cost for known count
    pub fn addSeqCost(
        self: *Evaluator,
        cost_kind: PerItemCost,
        n_items: usize,
        op_desc: OperationDesc,
    ) !void {
        const cost = try cost_kind.cost(n_items);
        try self.cost_accum.add(cost);

        if (self.cost_trace) |*trace| {
            try trace.append(.{
                .seq = .{
                    .op_desc = op_desc,
                    .cost_kind = cost_kind,
                    .n_items = n_items,
                },
            });
        }
    }

    /// Add type-based cost
    pub fn addTypeBasedCost(
        self: *Evaluator,
        cost_kind: TypeBasedCost,
        tpe: SType,
        op_desc: OperationDesc,
    ) !void {
        const cost = cost_kind.cost_fn(tpe);
        try self.cost_accum.add(cost);

        if (self.cost_trace) |*trace| {
            try trace.append(.{
                .type_based = .{
                    .op_desc = op_desc,
                    .cost_kind = cost_kind,
                    .tpe = tpe,
                },
            });
        }
    }
};
```

## PowHit (Autolykos2) Cost

Special cost computation for Autolykos2 mining[^13]:

```zig
const PowHitCost = struct {
    /// Cost of custom Autolykos2 hash function
    pub fn cost(
        k: u32,               // k-sum problem inputs
        msg: []const u8,      // message to hash
        nonce: []const u8,    // padding for PoW output
        h: []const u8,        // block height padding
    ) JitCost {
        const chunk_size = CalcBlake2b256.COST.chunk_size;
        const per_chunk = CalcBlake2b256.COST.per_chunk_cost.value;
        const base_cost: i32 = 500;

        // The heaviest part: k + 1 Blake2b256 invocations
        const input_len = msg.len + nonce.len + h.len;
        const chunks_per_hash = input_len / chunk_size + 1;
        const total_cost = base_cost + @as(i32, @intCast(k + 1)) *
            @as(i32, @intCast(chunks_per_hash)) * per_chunk;

        return .{ .value = total_cost };
    }
};
```

## Operation Cost Constants

Defined in operation companion structs:

```zig
const Constant = struct {
    pub const COST = FixedCost{ .cost = .{ .value = 5 } };
    // ...
};

const ValUse = struct {
    pub const COST = FixedCost{ .cost = .{ .value = 5 } };
    // ...
};

const If = struct {
    pub const COST = FixedCost{ .cost = .{ .value = 10 } };
    // ...
};

const MapCollection = struct {
    pub const COST = PerItemCost{
        .base_cost = .{ .value = 20 },
        .per_chunk_cost = .{ .value = 2 },
        .chunk_size = 10,
    };
    // ...
};

const CalcBlake2b256 = struct {
    pub const COST = PerItemCost{
        .base_cost = .{ .value = 20 },
        .per_chunk_cost = .{ .value = 7 },
        .chunk_size = 128,
    };
    // ...
};

const CalcSha256 = struct {
    pub const COST = PerItemCost{
        .base_cost = .{ .value = 80 },
        .per_chunk_cost = .{ .value = 8 },
        .chunk_size = 64,
    };
    // ...
};
```

## Cost Tracing Output

Example trace from evaluating a script:

```
Cost Trace
─────────────────────────────────────────────────────
Constant           :    5
ValUse             :    5
ByIndex            :   30
Constant           :    5
MapCollection[10]  :   22  (base=20, chunks=1)
Filter[5]          :   22  (base=20, chunks=1)
blake2b256[256]    :   34  (base=20, chunks=2)
─────────────────────────────────────────────────────
Total JitCost      :  123
Block Cost         :   12
```

## Complete Evaluation with Costing

```zig
pub fn evaluateWithCost(
    ergo_tree: *const ErgoTree,
    context: *const Context,
    cost_limit: JitCost,
    allocator: Allocator,
) !struct { result: SigmaBoolean, cost: JitCost } {
    var cost_accum = CostAccumulator.init(
        allocator,
        .{ .value = 0 },
        cost_limit,
    );

    var evaluator = Evaluator{
        .context = context,
        .constants = ergo_tree.constants,
        .cost_accum = cost_accum,
        .cost_trace = null,
        .profiler = null,
        .allocator = allocator,
    };

    const empty_env = Env.init(allocator);
    const result = try evaluator.eval(&empty_env, ergo_tree.root);

    const sigma_prop = result.asSigmaProp() orelse
        return error.NotSigmaProp;

    return .{
        .result = sigma_prop.sigma_boolean,
        .cost = evaluator.cost_accum.totalCost(),
    };
}
```

## Summary

This chapter covered the cost model that ensures all ErgoTree scripts terminate within bounded resources:

- **JitCost** uses 10x scaling from block costs, providing finer granularity for internal calculations while maintaining integer arithmetic without floating point
- **FixedCost** applies to constant-time operations like variable access (cost = 5) and conditionals (cost = 10)
- **PerItemCost** models operations that scale with input size using the formula: `baseCost + ceil(n/chunkSize) × perChunkCost`—this applies to collection operations and hash functions
- **TypeBasedCost** handles operations whose cost depends on operand type—BigInt operations are more expensive than primitive integer operations
- **CostAccumulator** tracks accumulated costs during evaluation and checks against the limit after each operation; exceeding the limit immediately fails evaluation
- **CostItem** types (`FixedCostItem`, `SeqCostItem`, `TypeBasedCostItem`) enable detailed cost tracing for debugging and optimization
- The **PowHit** cost function handles the special case of Autolykos2 mining operations

---

*Next: [Chapter 14: Verifier Implementation](./ch14-verifier-implementation.md)*

[^1]: Scala: [`JitCost.scala:3-7`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/JitCost.scala#L3-L7)

[^2]: Rust: [`cost_accum.rs:1-12`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval/cost_accum.rs#L1-L12)

[^3]: Scala: [`JitCost.scala:9-36`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/JitCost.scala#L9-L36)

[^4]: Scala: [`CostKind.scala:10-55`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/CostKind.scala#L10-L55)

[^5]: Rust: [`costs.rs:1-24`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval/costs.rs#L1-L24)

[^6]: Scala: [`CostKind.scala:60-66`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/CostKind.scala#L60-L66)

[^7]: Scala: [`CostItem.scala:3-78`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/CostItem.scala#L3-L78)

[^8]: Rust: [`cost_accum.rs:13-17`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval/cost_accum.rs#L13-L17)

[^9]: Scala: [`CostAccumulator.scala:7-79`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/interpreter/shared/src/main/scala/sigmastate/interpreter/CostAccumulator.scala#L7-L79)

[^10]: Rust: [`cost_accum.rs:19-43`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval/cost_accum.rs#L19-L43)

[^11]: Scala: [`ErgoTreeEvaluator.scala:18-86`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/eval/ErgoTreeEvaluator.scala#L18-L86)

[^12]: Rust: [`eval.rs:130-160`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-interpreter/src/eval.rs#L130-L160)

[^13]: Scala: [`CostKind.scala:71-88`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/CostKind.scala#L71-L88)