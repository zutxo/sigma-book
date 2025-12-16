# Chapter 12: Evaluation Model

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- AST structure ([Chapter 4](../part2/ch04-value-nodes.md))
- Operations ([Chapter 5](../part2/ch05-operations-opcodes.md))
- ErgoTree structure ([Chapter 3](../part1/ch03-ergotree-structure.md))

## Learning Objectives

- Understand direct-style big-step interpretation
- Implement eval dispatch for AST nodes
- Work with DataEnv for variable binding
- Track costs during evaluation

## Evaluation Architecture

The interpreter uses direct-style big-step evaluation[^1][^2]:

```
Evaluation Flow
─────────────────────────────────────────────────────

┌──────────────────────────────────────────────────┐
│              ErgoTreeEvaluator                   │
├──────────────────────────────────────────────────┤
│  context: Context     (SELF, INPUTS, OUTPUTS)    │
│  constants: []Const   (segregated constants)     │
│  cost_accum: CostAcc  (tracks execution cost)    │
│  env: Env             (variable bindings)        │
└───────────────────────┬──────────────────────────┘
                        │
                        │ eval(expr)
                        ▼
┌──────────────────────────────────────────────────┐
│              AST Traversal                       │
│                                                  │
│    Expr.eval(env, ctx)                           │
│         │                                        │
│         ├── Evaluate children                    │
│         ├── Add operation cost                   │
│         ├── Perform operation                    │
│         └── Return result Value                  │
└──────────────────────────────────────────────────┘
```

## Evaluator Structure

```zig
const Evaluator = struct {
    context: *const Context,
    constants: []const Constant,
    cost_accum: CostAccumulator,
    allocator: Allocator,

    pub fn init(
        context: *const Context,
        constants: []const Constant,
        cost_limit: JitCost,
        allocator: Allocator,
    ) Evaluator {
        return .{
            .context = context,
            .constants = constants,
            .cost_accum = CostAccumulator.init(cost_limit),
            .allocator = allocator,
        };
    }

    /// Evaluate expression in given environment
    pub fn eval(self: *Evaluator, env: *const Env, expr: *const Expr) !Value {
        return expr.eval(env, self);
    }

    /// Evaluate to specific type
    pub fn evalTo(
        self: *Evaluator,
        comptime T: type,
        env: *const Env,
        expr: *const Expr,
    ) !T {
        const result = try self.eval(env, expr);
        return result.as(T) orelse error.TypeMismatch;
    }

    /// Add fixed cost
    pub fn addCost(self: *Evaluator, cost: FixedCost, op: OpCode) !void {
        try self.cost_accum.add(cost.value, op);
    }

    /// Add per-item cost
    pub fn addSeqCost(self: *Evaluator, cost: PerItemCost, n_items: usize, op: OpCode) !void {
        const total = cost.base.value + (n_items / cost.chunk_size + 1) * cost.per_chunk.value;
        try self.cost_accum.add(total, op);
    }
};
```

## Environment (Variable Binding)

The `Env` maps variable IDs to computed values[^3][^4]:

```zig
const Env = struct {
    /// HashMap from variable ID to value
    bindings: std.AutoHashMap(u32, Value),
    allocator: Allocator,

    pub fn init(allocator: Allocator) Env {
        return .{
            .bindings = std.AutoHashMap(u32, Value).init(allocator),
            .allocator = allocator,
        };
    }

    /// Look up variable by ID
    pub fn get(self: *const Env, val_id: u32) ?Value {
        return self.bindings.get(val_id);
    }

    /// Create new environment with additional binding
    /// NOTE: This implementation clones the HashMap on every extend() call.
    /// In production, use a pre-allocated binding stack with O(1) extend/pop:
    ///   bindings: [MAX_BINDINGS]Binding (pre-allocated)
    ///   stack_ptr: usize (grows/shrinks without allocation)
    /// See ZIGMA_STYLE.md for zero-allocation evaluation patterns.
    pub fn extend(self: *const Env, val_id: u32, value: Value) !Env {
        var new_env = Env{
            .bindings = try self.bindings.clone(),
            .allocator = self.allocator,
        };
        try new_env.bindings.put(val_id, value);
        return new_env;
    }

    /// Create new environment with multiple bindings
    pub fn extendMany(self: *const Env, bindings: []const struct { id: u32, val: Value }) !Env {
        var new_env = Env{
            .bindings = try self.bindings.clone(),
            .allocator = self.allocator,
        };
        for (bindings) |b| {
            try new_env.bindings.put(b.id, b.val);
        }
        return new_env;
    }
};
```

## Expression Dispatch

Each expression type implements eval[^5][^6]:

```zig
const Expr = union(enum) {
    constant: Constant,
    const_placeholder: ConstantPlaceholder,
    val_use: ValUse,
    block_value: BlockValue,
    func_value: FuncValue,
    apply: Apply,
    if_op: If,
    bin_op: BinOp,
    // ... other expression types

    /// Evaluate expression recursively
    /// NOTE: This recursive approach is clear for learning but uses the call
    /// stack. In production, use an explicit work stack to:
    /// 1. Guarantee bounded stack depth (no stack overflow)
    /// 2. Enable O(1) reset between transactions
    /// See ZIGMA_STYLE.md for iterative evaluation patterns.
    pub fn eval(self: *const Expr, env: *const Env, E: *Evaluator) !Value {
        return switch (self.*) {
            .constant => |c| c.eval(env, E),
            .const_placeholder => |cp| cp.eval(env, E),
            .val_use => |vu| vu.eval(env, E),
            .block_value => |bv| bv.eval(env, E),
            .func_value => |fv| fv.eval(env, E),
            .apply => |a| a.eval(env, E),
            .if_op => |i| i.eval(env, E),
            .bin_op => |b| b.eval(env, E),
            // ... dispatch to other eval implementations
        };
    }
};
```

## Constant Evaluation

Constants return their value with fixed cost[^7]:

```zig
const Constant = struct {
    tpe: SType,
    value: Literal,

    pub const COST = FixedCost{ .value = 5 };

    pub fn eval(self: *const Constant, _: *const Env, E: *Evaluator) !Value {
        try E.addCost(COST, OpCode.Constant);
        return Value.fromLiteral(self.value);
    }
};

const ConstantPlaceholder = struct {
    index: u32,
    tpe: SType,

    pub const COST = FixedCost{ .value = 1 };

    pub fn eval(self: *const ConstantPlaceholder, _: *const Env, E: *Evaluator) !Value {
        try E.addCost(COST, OpCode.ConstantPlaceholder);
        if (self.index >= E.constants.len) {
            return error.IndexOutOfBounds;
        }
        const c = E.constants[self.index];
        return Value.fromLiteral(c.value);
    }
};
```

## Variable Access

ValUse looks up variables in environment[^8]:

```zig
const ValUse = struct {
    val_id: u32,
    tpe: SType,

    pub const COST = FixedCost{ .value = 5 };

    pub fn eval(self: *const ValUse, env: *const Env, E: *Evaluator) !Value {
        try E.addCost(COST, OpCode.ValUse);
        return env.get(self.val_id) orelse error.UndefinedVariable;
    }
};
```

## Block Evaluation

Blocks introduce variable bindings[^9]:

```zig
const BlockValue = struct {
    items: []const ValDef,
    result: *const Expr,

    pub const COST = PerItemCost{
        .base = JitCost{ .value = 1 },
        .per_chunk = JitCost{ .value = 1 },
        .chunk_size = 1,
    };

    pub fn eval(self: *const BlockValue, env: *const Env, E: *Evaluator) !Value {
        try E.addSeqCost(COST, self.items.len, OpCode.BlockValue);

        var cur_env = env.*;
        for (self.items) |item| {
            // Evaluate right-hand side
            const rhs_val = try item.rhs.eval(&cur_env, E);

            // Extend environment with new binding
            try E.addCost(FuncValue.ADD_TO_ENV_COST, OpCode.FuncValue);
            cur_env = try cur_env.extend(item.id, rhs_val);
        }

        // Evaluate result in extended environment
        return self.result.eval(&cur_env, E);
    }
};

const ValDef = struct {
    id: u32,
    tpe: SType,
    rhs: *const Expr,
};
```

## Lambda Functions

FuncValue creates closures[^10]:

```zig
const FuncValue = struct {
    args: []const FuncArg,
    body: *const Expr,

    pub const COST = FixedCost{ .value = 10 };
    pub const ADD_TO_ENV_COST = FixedCost{ .value = 5 };

    pub fn eval(self: *const FuncValue, env: *const Env, E: *Evaluator) !Value {
        try E.addCost(COST, OpCode.FuncValue);

        // Create closure capturing current environment
        return Value{
            .closure = .{
                .captured_env = env.*,
                .args = self.args,
                .body = self.body,
            },
        };
    }
};

const FuncArg = struct {
    id: u32,
    tpe: SType,
};

const Apply = struct {
    func: *const Expr,
    args: *const Expr,

    pub fn eval(self: *const Apply, env: *const Env, E: *Evaluator) !Value {
        // Evaluate function
        const func_val = try self.func.eval(env, E);
        const closure = func_val.closure;

        // Evaluate argument
        const arg_val = try self.args.eval(env, E);

        // Extend closure's captured env with argument binding
        try E.addCost(FuncValue.ADD_TO_ENV_COST, OpCode.Apply);
        var new_env = try closure.captured_env.extend(closure.args[0].id, arg_val);

        // Evaluate body in new environment
        return closure.body.eval(&new_env, E);
    }
};
```

## Conditional Evaluation

If uses short-circuit semantics[^11]:

```zig
const If = struct {
    condition: *const Expr,
    true_branch: *const Expr,
    false_branch: *const Expr,

    pub const COST = FixedCost{ .value = 10 };

    pub fn eval(self: *const If, env: *const Env, E: *Evaluator) !Value {
        // Evaluate condition
        const cond = try E.evalTo(bool, env, self.condition);

        try E.addCost(COST, OpCode.If);

        // Only evaluate taken branch (short-circuit)
        if (cond) {
            return self.true_branch.eval(env, E);
        } else {
            return self.false_branch.eval(env, E);
        }
    }
};
```

## Collection Operations

Map, filter, fold evaluate with per-item costs[^12]:

```zig
const Map = struct {
    input: *const Expr,
    mapper: *const Expr,

    pub const COST = PerItemCost{
        .base = JitCost{ .value = 10 },
        .per_chunk = JitCost{ .value = 5 },
        .chunk_size = 10,
    };

    pub fn eval(self: *const Map, env: *const Env, E: *Evaluator) !Value {
        const input_coll = try E.evalTo(Collection, env, self.input);
        const mapper_fn = try E.evalTo(Closure, env, self.mapper);

        try E.addSeqCost(COST, input_coll.len, OpCode.Map);

        var result = try E.allocator.alloc(Value, input_coll.len);
        for (input_coll.items, 0..) |item, i| {
            // Apply mapper to each element
            try E.addCost(FuncValue.ADD_TO_ENV_COST, OpCode.Map);
            var fn_env = try mapper_fn.captured_env.extend(mapper_fn.args[0].id, item);
            result[i] = try mapper_fn.body.eval(&fn_env, E);
        }

        return Value{ .coll = .{ .items = result } };
    }
};

const Fold = struct {
    input: *const Expr,
    zero: *const Expr,
    folder: *const Expr,

    pub const COST = PerItemCost{
        .base = JitCost{ .value = 10 },
        .per_chunk = JitCost{ .value = 5 },
        .chunk_size = 10,
    };

    pub fn eval(self: *const Fold, env: *const Env, E: *Evaluator) !Value {
        const input_coll = try E.evalTo(Collection, env, self.input);
        const zero_val = try self.zero.eval(env, E);
        const folder_fn = try E.evalTo(Closure, env, self.folder);

        try E.addSeqCost(COST, input_coll.len, OpCode.Fold);

        var accum = zero_val;
        for (input_coll.items) |item| {
            // folder takes (accum, item)
            const tuple = Value{ .tuple = .{ accum, item } };
            try E.addCost(FuncValue.ADD_TO_ENV_COST, OpCode.Fold);
            var fn_env = try folder_fn.captured_env.extend(folder_fn.args[0].id, tuple);
            accum = try folder_fn.body.eval(&fn_env, E);
        }

        return accum;
    }
};
```

## Binary Operations

```zig
const BinOp = struct {
    kind: Kind,
    left: *const Expr,
    right: *const Expr,

    const Kind = enum {
        plus, minus, multiply, divide, modulo,
        gt, ge, lt, le, eq, neq,
        bin_and, bin_or, bin_xor,
    };

    pub fn eval(self: *const BinOp, env: *const Env, E: *Evaluator) !Value {
        const left_val = try self.left.eval(env, E);
        const right_val = try self.right.eval(env, E);

        return switch (self.kind) {
            .plus => try evalPlus(left_val, right_val, E),
            .minus => try evalMinus(left_val, right_val, E),
            .gt => try evalGt(left_val, right_val, E),
            // ... other operations
        };
    }

    fn evalPlus(left: Value, right: Value, E: *Evaluator) !Value {
        try E.addCost(ArithOp.PLUS_COST, OpCode.Plus);
        return switch (left) {
            .int => |l| Value{ .int = try std.math.add(i32, l, right.int) },
            .long => |l| Value{ .long = try std.math.add(i64, l, right.long) },
            else => error.TypeMismatch,
        };
    }
};
```

## Top-Level Evaluation

Reduce ErgoTree to SigmaBoolean:

```zig
pub fn reduceToSigmaBoolean(
    ergo_tree: *const ErgoTree,
    context: *const Context,
    cost_limit: JitCost,
    allocator: Allocator,
) !struct { prop: SigmaBoolean, cost: JitCost } {
    var evaluator = Evaluator.init(
        context,
        ergo_tree.constants,
        cost_limit,
        allocator,
    );

    const empty_env = Env.init(allocator);
    const result = try evaluator.eval(&empty_env, ergo_tree.root);

    const sigma_prop = result.asSigmaProp() orelse
        return error.NotSigmaProp;

    return .{
        .prop = sigma_prop.sigma_boolean,
        .cost = evaluator.cost_accum.totalCost(),
    };
}
```

## Summary

- **Direct-style** big-step interpreter evaluates expressions recursively
- **Env** maps variable IDs to values; immutable with functional updates
- **Each node** implements `eval()` returning Value and accumulating cost
- **BlockValue** extends environment with ValDef bindings
- **FuncValue** creates closures capturing the environment
- **If** uses short-circuit evaluation (only one branch executed)
- **Collection ops** (map, filter, fold) have per-item costs
- Top-level reduces to **SigmaBoolean** for cryptographic verification

---

*Next: [Chapter 13: Cost Model](./ch13-cost-model.md)*

[^1]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/CErgoTreeEvaluator.scala`

[^2]: Rust: `ergotree-interpreter/src/eval.rs:1-100`

[^3]: Scala: `data/shared/src/main/scala/sigma/eval/ErgoTreeEvaluator.scala` (DataEnv)

[^4]: Rust: `ergotree-interpreter/src/eval/env.rs`

[^5]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala` (eval methods)

[^6]: Rust: `ergotree-interpreter/src/eval/expr.rs:14-100`

[^7]: Scala: `data/shared/src/main/scala/sigma/ast/values.scala` (ConstantNode.eval)

[^8]: Rust: `ergotree-interpreter/src/eval/val_use.rs`

[^9]: Rust: `ergotree-interpreter/src/eval/block.rs`

[^10]: Rust: `ergotree-interpreter/src/eval/func_value.rs`

[^11]: Rust: `ergotree-interpreter/src/eval/if_op.rs`

[^12]: Rust: `ergotree-interpreter/src/eval/coll_map.rs`, `coll_fold.rs`