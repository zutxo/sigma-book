# Chapter 18: Intermediate Representation (IR)

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 17](./ch17-semantic-analysis.md) for the typed AST that feeds into IR construction
- [Chapter 5](../part2/ch05-operations-opcodes.md) for operation codes that IR nodes map to
- Understanding of compiler optimization concepts: CSE, dead code elimination

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the graph-based IR design using the Def/Ref pattern
- Implement common subexpression elimination (CSE) via hash-consing
- Apply graph rewriting for algebraic simplifications
- Trace the AST → Graph IR → Optimized Tree transformations

## IR Architecture Overview

The Scala compiler uses a sophisticated graph-based IR for optimization[^1][^2]. The Rust compiler uses a simpler direct HIR→MIR pipeline[^3].

```
Compilation Pipelines
─────────────────────────────────────────────────────

Scala (Graph IR):
┌─────────┐   GraphBuilding   ┌──────────┐   TreeBuilding    ┌──────────┐
│ Typed   │ ─────────────────>│ Graph IR │ ─────────────────>│ Optimized│
│ AST     │   (+ CSE)         │ (Def/Ref)│   (ValDef min)    │ ErgoTree │
└─────────┘                   └──────────┘                   └──────────┘
                                   │
                                   │ DefRewriting
                                   │ (algebraic simplifications)
                                   ▼

Rust (Direct):
┌─────────┐   Lower    ┌──────────┐   Lower    ┌──────────┐   Check   ┌──────────┐
│ HIR     │ ─────────> │ Bound    │ ─────────> │ Typed    │ ─────────>│ MIR/     │
│ (parse) │            │ HIR      │            │ HIR      │           │ ErgoTree │
└─────────┘            └──────────┘            └──────────┘           └──────────┘
```

## The Def/Ref Pattern

The core IR abstraction uses definitions (nodes) and references (edges)[^4][^5]:

```zig
/// Reference to a definition (graph edge)
/// Like a pointer but with type information
const Sym = u32;  // Symbol ID

/// Type descriptor for IR values
const Elem = struct {
    stype: SType,
    source_type: ?*const std.meta.Type,
};

/// Base type for all graph nodes
const Node = struct {
    /// Unique ID assigned on creation
    node_id: u32,
    /// Cached dependencies (other nodes this one uses)
    deps: ?[]const Sym,
    /// Cached hash for structural equality
    hash_code: u32,

    pub fn getDeps(self: *const Node) []const Sym {
        if (self.deps) |d| return d;
        // Computed lazily from node contents
        return computeDeps(self);
    }
};

/// Definition of a computation (graph node)
const Def = struct {
    node: Node,
    /// Type of the result value
    result_type: Elem,
    /// Reference to this definition (created lazily)
    self_ref: ?Sym,

    pub fn self(d: *Def, ctx: *IRContext) Sym {
        if (d.self_ref) |s| return s;
        const sym = ctx.freshSym(d);
        d.self_ref = sym;
        return sym;
    }
};
```

## IR Context

The IR context manages the graph and provides CSE[^6][^7]:

```zig
const IRContext = struct {
    allocator: Allocator,
    /// Counter for unique node IDs
    id_counter: u32,
    /// Global definitions: Def hash → Sym
    /// This enables CSE through hash-consing
    global_defs: std.HashMap(*const Def, Sym, DefHashContext, 80),
    /// Sym → Def mapping
    sym_to_def: std.AutoHashMap(Sym, *const Def),

    pub fn init(allocator: Allocator) IRContext {
        return .{
            .allocator = allocator,
            .id_counter = 0,
            .global_defs = std.HashMap(*const Def, Sym, DefHashContext, 80).init(allocator),
            .sym_to_def = std.AutoHashMap(Sym, *const Def).init(allocator),
        };
    }

    /// Generate fresh symbol ID
    pub fn freshSym(self: *IRContext, def: *const Def) Sym {
        const id = self.id_counter;
        self.id_counter += 1;
        self.sym_to_def.put(id, def) catch unreachable;
        return id;
    }

    /// Create or reuse existing definition (CSE)
    pub fn reifyObject(self: *IRContext, d: *Def) Sym {
        return self.findOrCreateDefinition(d);
    }

    /// Hash-consing: lookup by structural equality
    fn findOrCreateDefinition(self: *IRContext, d: *Def) Sym {
        if (self.global_defs.get(d)) |existing_sym| {
            // Reuse existing definition
            return existing_sym;
        }
        // Register new definition
        const sym = d.self(self);
        self.global_defs.put(d, sym) catch unreachable;
        return sym;
    }
};

/// Hash context for structural equality of definitions
const DefHashContext = struct {
    pub fn hash(_: DefHashContext, def: *const Def) u64 {
        // Hash based on node type and contents (structural)
        return def.node.hash_code;
    }

    pub fn eql(_: DefHashContext, a: *const Def, b: *const Def) bool {
        // Structural equality of definitions
        return structuralEqual(a, b);
    }
};
```

## Common Subexpression Elimination

CSE is achieved automatically through hash-consing[^8]:

```
CSE Through Hash-Consing
─────────────────────────────────────────────────────

Source:
  val a = SELF.value
  val b = SELF.value  // Same computation!
  a + b

Step 1: Build graph for SELF.value
  s1 = Self
  s2 = MethodCall(s1, "value")  → stored in global_defs

Step 2: Build graph for second SELF.value
  s1 = Self                      → already exists, reuse
  s2 = MethodCall(s1, "value")   → lookup in global_defs
                                 → found! return existing s2

Step 3: Build addition
  s3 = Plus(s2, s2)              → both operands point to s2

Result: Single computation of SELF.value
```

```zig
/// Build graph from typed AST
const GraphBuilder = struct {
    ctx: *IRContext,
    env: std.StringHashMap(Sym),

    pub fn buildGraph(self: *GraphBuilder, expr: *const TypedExpr) !Sym {
        return switch (expr.kind) {
            .constant => |c| self.buildConstant(c),
            .val_use => |name| self.env.get(name) orelse error.UndefinedVariable,
            .block => |b| self.buildBlock(b),
            .bin_op => |op| self.buildBinOp(op),
            .method_call => |mc| self.buildMethodCall(mc),
            .if_expr => |i| self.buildIf(i),
            .func_value => |f| self.buildLambda(f),
            .apply => |a| self.buildApply(a),
        };
    }

    fn buildConstant(self: *GraphBuilder, c: Constant) Sym {
        const def = self.ctx.allocator.create(ConstDef) catch unreachable;
        def.* = .{ .value = c };
        // CSE: if same constant exists, reuse it
        return self.ctx.reifyObject(&def.base);
    }

    fn buildBinOp(self: *GraphBuilder, op: *const BinOp) !Sym {
        const left_sym = try self.buildGraph(op.left);
        const right_sym = try self.buildGraph(op.right);

        const def = self.ctx.allocator.create(BinOpDef) catch unreachable;
        def.* = .{
            .op = op.kind,
            .left = left_sym,
            .right = right_sym,
        };
        // CSE: reuse if same operation on same operands exists
        return self.ctx.reifyObject(&def.base);
    }

    fn buildMethodCall(self: *GraphBuilder, mc: *const MethodCall) !Sym {
        const receiver_sym = try self.buildGraph(mc.receiver);
        var arg_syms = try self.ctx.allocator.alloc(Sym, mc.args.len);
        for (mc.args, 0..) |arg, i| {
            arg_syms[i] = try self.buildGraph(arg);
        }

        const def = self.ctx.allocator.create(MethodCallDef) catch unreachable;
        def.* = .{
            .receiver = receiver_sym,
            .method = mc.method,
            .args = arg_syms,
        };
        // CSE: reuse if identical method call exists
        return self.ctx.reifyObject(&def.base);
    }
};
```

## Graph Rewriting

Algebraic simplifications are applied as rewrite rules[^9][^10]:

```zig
/// Rewriting rules for optimization
const DefRewriter = struct {
    ctx: *IRContext,

    /// Called on each new definition
    /// Returns replacement Sym or null for no rewrite
    pub fn rewriteDef(self: *DefRewriter, d: *const Def) ?Sym {
        return switch (d.kind()) {
            .coll_length => self.rewriteLength(d.as(CollLengthDef)),
            .coll_map => self.rewriteMap(d.as(CollMapDef)),
            .coll_zip => self.rewriteZip(d.as(CollZipDef)),
            .option_get_or_else => self.rewriteGetOrElse(d.as(OptionGetOrElseDef)),
            else => null,
        };
    }

    /// xs.map(f).length => xs.length
    fn rewriteLength(self: *DefRewriter, len_def: *const CollLengthDef) ?Sym {
        const input = self.ctx.getDef(len_def.input);
        return switch (input.kind()) {
            .coll_map => |map_def| {
                // Rule: xs.map(f).length => xs.length
                return self.makeLength(map_def.input);
            },
            .coll_replicate => |rep_def| {
                // Rule: replicate(len, v).length => len
                return rep_def.length;
            },
            .const_coll => |coll_def| {
                // Rule: Const(coll).length => coll.length
                return self.makeConstant(.{ .int = @intCast(coll_def.items.len) });
            },
            .coll_from_items => |items_def| {
                // Rule: Coll(items).length => items.length
                return self.makeConstant(.{ .int = @intCast(items_def.items.len) });
            },
            else => null,
        };
    }

    /// xs.map(identity) => xs
    /// xs.map(f).map(g) => xs.map(x => g(f(x)))
    fn rewriteMap(self: *DefRewriter, map_def: *const CollMapDef) ?Sym {
        const mapper = self.ctx.getDef(map_def.mapper);

        // Rule: xs.map(identity) => xs
        if (isIdentityLambda(mapper)) {
            return map_def.input;
        }

        const input = self.ctx.getDef(map_def.input);
        return switch (input.kind()) {
            .coll_replicate => |rep_def| {
                // Rule: replicate(l, v).map(f) => replicate(l, f(v))
                const applied = self.makeApply(map_def.mapper, rep_def.value);
                return self.makeReplicate(rep_def.length, applied);
            },
            .coll_map => |inner_map| {
                // Rule: xs.map(f).map(g) => xs.map(x => g(f(x)))
                const composed = self.composeLambdas(inner_map.mapper, map_def.mapper);
                return self.makeMap(inner_map.input, composed);
            },
            else => null,
        };
    }

    /// replicate(l, x).zip(replicate(l, y)) => replicate(l, (x, y))
    fn rewriteZip(self: *DefRewriter, zip_def: *const CollZipDef) ?Sym {
        const left = self.ctx.getDef(zip_def.left);
        const right = self.ctx.getDef(zip_def.right);

        if (left.kind() == .coll_replicate and right.kind() == .coll_replicate) {
            const rep_l = left.as(CollReplicateDef);
            const rep_r = right.as(CollReplicateDef);

            // Check same length and builder
            if (rep_l.length == rep_r.length and rep_l.builder == rep_r.builder) {
                const pair = self.makePair(rep_l.value, rep_r.value);
                return self.makeReplicate(rep_l.length, pair);
            }
        }
        return null;
    }

    /// Some(x).getOrElse(d) => x
    fn rewriteGetOrElse(self: *DefRewriter, def: *const OptionGetOrElseDef) ?Sym {
        const opt = self.ctx.getDef(def.option);
        if (opt.kind() == .option_const) {
            const opt_const = opt.as(OptionConstDef);
            if (opt_const.value) |v| {
                return self.liftValue(v);
            }
        }
        return null;
    }
};
```

## Sigma-Specific Rewrites

Special optimizations for Sigma propositions[^11]:

```zig
/// Sigma-specific rewriting rules
const SigmaRewriter = struct {
    ctx: *IRContext,

    pub fn rewriteSigma(self: *SigmaRewriter, d: *const Def) ?Sym {
        return switch (d.kind()) {
            .sigma_prop_is_valid => self.rewriteIsValid(d),
            .sigma_prop_from_bool => self.rewriteSigmaProp(d),
            .all_of => self.rewriteAllOf(d),
            .any_of => self.rewriteAnyOf(d),
            else => null,
        };
    }

    /// sigmaProp(sp.isValid) => sp
    fn rewriteIsValid(self: *SigmaRewriter, d: *const Def) ?Sym {
        const is_valid = d.as(SigmaIsValidDef);
        const inner = self.ctx.getDef(is_valid.prop);

        if (inner.kind() == .sigma_prop_from_bool) {
            const from_bool = inner.as(SigmaPropFromBoolDef);
            // Check if the bool is another isValid
            const bool_def = self.ctx.getDef(from_bool.bool_expr);
            if (bool_def.kind() == .sigma_prop_is_valid) {
                return bool_def.as(SigmaIsValidDef).prop;
            }
        }
        return null;
    }

    /// sigmaProp(b).isValid => b
    fn rewriteSigmaProp(self: *SigmaRewriter, d: *const Def) ?Sym {
        _ = d;
        _ = self;
        // This rewrite is handled in rewriteIsValid
        return null;
    }

    /// allOf(Coll(b1, ..., sp1.isValid, ...)) =>
    ///   (allOf(Coll(b1, ...)) && allZK(sp1, ...)).isValid
    fn rewriteAllOf(self: *SigmaRewriter, d: *const Def) ?Sym {
        const all_of = d.as(AllOfDef);
        const items = self.extractItems(all_of.input) orelse return null;

        var bools = std.ArrayList(Sym).init(self.ctx.allocator);
        var sigmas = std.ArrayList(Sym).init(self.ctx.allocator);

        for (items) |item| {
            const item_def = self.ctx.getDef(item);
            if (item_def.kind() == .sigma_prop_is_valid) {
                const is_valid = item_def.as(SigmaIsValidDef);
                sigmas.append(is_valid.prop) catch unreachable;
            } else {
                bools.append(item) catch unreachable;
            }
        }

        if (sigmas.items.len == 0) return null;

        // Build: (allOf(bools) && allZK(sigmas)).isValid
        const zk_all = self.makeAllZK(sigmas.items);
        if (bools.items.len == 0) {
            return self.makeIsValid(zk_all);
        }
        const bool_all = self.makeSigmaProp(self.makeAllOf(bools.items));
        const combined = self.makeSigmaAnd(bool_all, zk_all);
        return self.makeIsValid(combined);
    }
};
```

## Tree Building

Transform optimized graph back to ErgoTree[^12][^13]:

```zig
/// Transform graph IR to ErgoTree
const TreeBuilder = struct {
    ctx: *IRContext,
    /// Maps symbols to ValDef IDs
    env: std.AutoHashMap(Sym, struct { id: u32, tpe: SType }),
    /// Current ValDef ID counter
    def_id: u32,
    allocator: Allocator,

    pub fn buildTree(self: *TreeBuilder, root: Sym) !*Expr {
        // Compute usage counts to minimize ValDefs
        const usage = self.computeUsageCounts(root);

        // Build topological schedule
        const schedule = self.buildSchedule(root);

        // Process nodes, introducing ValDefs only for multi-use
        var val_defs = std.ArrayList(ValDef).init(self.allocator);
        for (schedule) |sym| {
            if (usage.get(sym).? > 1) {
                // Multi-use node: create ValDef
                const rhs = try self.buildValue(sym);
                const tpe = self.ctx.getDef(sym).result_type.stype;
                try val_defs.append(.{
                    .id = self.def_id,
                    .tpe = tpe,
                    .rhs = rhs,
                });
                try self.env.put(sym, .{ .id = self.def_id, .tpe = tpe });
                self.def_id += 1;
            }
        }

        // Build result expression
        const result = try self.buildValue(root);

        // Wrap in block if we have ValDefs
        if (val_defs.items.len == 0) {
            return result;
        }
        return self.makeBlock(val_defs.items, result);
    }

    fn buildValue(self: *TreeBuilder, sym: Sym) !*Expr {
        // Check if already bound in environment
        if (self.env.get(sym)) |binding| {
            return self.makeValUse(binding.id, binding.tpe);
        }

        const def = self.ctx.getDef(sym);
        return switch (def.kind()) {
            .constant => |c| self.makeConstant(c),
            .context_prop => |prop| self.buildContextProp(prop),
            .method_call => |mc| self.buildMethodCall(mc),
            .bin_op => |op| self.buildBinOp(op),
            .lambda => |lam| self.buildLambda(lam),
            .apply => |app| self.buildApply(app),
            .if_then_else => |ite| self.buildIf(ite),
            else => error.UnhandledDefKind,
        };
    }

    fn computeUsageCounts(self: *TreeBuilder, root: Sym) std.AutoHashMap(Sym, u32) {
        var counts = std.AutoHashMap(Sym, u32).init(self.allocator);
        self.countUsagesRecursive(root, &counts);
        return counts;
    }

    fn countUsagesRecursive(self: *TreeBuilder, sym: Sym, counts: *std.AutoHashMap(Sym, u32)) void {
        const current = counts.get(sym) orelse 0;
        counts.put(sym, current + 1) catch unreachable;

        // Only traverse dependencies on first visit
        if (current == 0) {
            const def = self.ctx.getDef(sym);
            for (def.node.getDeps()) |dep| {
                self.countUsagesRecursive(dep, counts);
            }
        }
    }

    fn buildSchedule(self: *TreeBuilder, root: Sym) []const Sym {
        // Topological sort via DFS
        var visited = std.AutoHashMap(Sym, void).init(self.allocator);
        var schedule = std.ArrayList(Sym).init(self.allocator);
        self.dfs(root, &visited, &schedule);
        return schedule.items;
    }

    fn dfs(self: *TreeBuilder, sym: Sym, visited: *std.AutoHashMap(Sym, void), schedule: *std.ArrayList(Sym)) void {
        if (visited.contains(sym)) return;
        visited.put(sym, {}) catch unreachable;

        const def = self.ctx.getDef(sym);
        for (def.node.getDeps()) |dep| {
            self.dfs(dep, visited, schedule);
        }
        schedule.append(sym) catch unreachable;
    }
};
```

## Operation Translation

Map IR operations to ErgoTree nodes[^14]:

```zig
/// Recognize arithmetic operations
fn translateArithOp(op: BinOpKind) ?OpCode {
    return switch (op) {
        .plus => OpCode.Plus,
        .minus => OpCode.Minus,
        .multiply => OpCode.Multiply,
        .divide => OpCode.Division,
        .modulo => OpCode.Modulo,
        .min => OpCode.Min,
        .max => OpCode.Max,
        else => null,
    };
}

/// Recognize comparison operations
fn translateRelationOp(op: BinOpKind) ?fn (*Expr, *Expr) *Expr {
    return switch (op) {
        .eq => makeEQ,
        .neq => makeNEQ,
        .gt => makeGT,
        .lt => makeLT,
        .ge => makeGE,
        .le => makeLE,
        else => null,
    };
}

/// Recognize context properties
fn translateContextProp(prop: ContextProperty) *Expr {
    return switch (prop) {
        .height => &expr_height,
        .inputs => &expr_inputs,
        .outputs => &expr_outputs,
        .self => &expr_self,
    };
}

/// Internal definitions should not become ValDefs
fn isInternalDef(def: *const Def) bool {
    return switch (def.kind()) {
        .sigma_dsl_builder, .coll_builder => true,
        else => false,
    };
}
```

## Rust HIR (Alternative Approach)

The Rust compiler uses a simpler tree-based HIR without graph IR[^15][^16]:

```zig
/// Rust-style HIR expression
const HirExpr = struct {
    kind: ExprKind,
    span: TextRange,
    tpe: ?SType,

    const ExprKind = union(enum) {
        ident: []const u8,
        binary: Binary,
        global_vars: GlobalVars,
        literal: Literal,
    };

    const Binary = struct {
        op: Spanned(BinaryOp),
        lhs: *HirExpr,
        rhs: *HirExpr,
    };

    const GlobalVars = enum {
        height,
    };

    const Literal = union(enum) {
        int: i32,
        long: i64,
    };
};

/// Rewrite HIR expressions (simpler than graph rewriting)
fn rewrite(
    e: HirExpr,
    f: fn (*const HirExpr) ?HirExpr,
) HirExpr {
    // Apply rewrite function
    const rewritten = f(&e) orelse e;

    // Recursively rewrite children
    return switch (rewritten.kind) {
        .binary => |bin| blk: {
            const new_lhs = f(bin.lhs) orelse bin.lhs.*;
            const new_rhs = f(bin.rhs) orelse bin.rhs.*;
            break :blk HirExpr{
                .kind = .{ .binary = .{
                    .op = bin.op,
                    .lhs = &new_lhs,
                    .rhs = &new_rhs,
                }},
                .span = rewritten.span,
                .tpe = rewritten.tpe,
            };
        },
        else => rewritten,
    };
}
```

## CSE Example Walkthrough

```
Source:
─────────────────────────────────────────────────────
{
  val x = SELF.value
  val y = SELF.value    // Duplicate!
  val z = OUTPUTS(0).value
  x + y > z
}

After GraphBuilding (with CSE):
─────────────────────────────────────────────────────
s1 = Context.SELF
s2 = s1.value           // Single node for both x and y
s3 = Context.OUTPUTS
s4 = s3.apply(0)
s5 = s4.value
s6 = Plus(s2, s2)       // x + y = s2 + s2
s7 = GT(s6, s5)

After TreeBuilding (ValDef minimization):
─────────────────────────────────────────────────────
{
  val v1 = SELF.value   // s2 used twice → ValDef
  GT(Plus(v1, v1), OUTPUTS(0).value)
}

Nodes s1, s3, s4, s5 have single use → inlined
Node s2 has multiple uses → ValDef introduced
```

## Summary

- **Def/Ref pattern** separates computation definitions from references
- **Hash-consing** enables automatic CSE—structurally equal nodes share identity
- **Graph rewriting** applies algebraic simplifications (map fusion, etc.)
- **TreeBuilding** transforms graph back to ErgoTree with minimal ValDefs
- **Usage counting** determines which nodes need ValDef bindings
- Scala uses full graph IR; Rust uses simpler tree-based HIR
- IR optimizations reduce serialized ErgoTree size
- Not part of consensus—compiler-only optimization

---

*Next: [Chapter 19: Compiler Pipeline](./ch19-compiler-pipeline.md)*

[^1]: Scala: [`IRContext.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/IRContext.scala)

[^2]: Scala: [`Base.scala:17-200`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/Base.scala#L17-L200) (Node, Def, Ref)

[^3]: Rust: [`compiler.rs:59-76`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/compiler.rs#L59-L76) (compile pipeline)

[^4]: Scala: [`Base.scala:100-160`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/Base.scala#L100-L160) (Def trait)

[^5]: Rust: [`hir.rs:32-37`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/hir.rs#L32-L37) (Expr struct)

[^6]: Scala: [`IRContext.scala:28-50`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/IRContext.scala#L28-L50) (cake pattern)

[^7]: Rust: [`compiler.rs:78-87`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/compiler.rs#L78-L87) (compile_hir)

[^8]: Scala: [`GraphBuilding.scala:28-35`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/GraphBuilding.scala#L28-L35) (CSE documentation)

[^9]: Scala: [`IRContext.scala:105-150`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/IRContext.scala#L105-L150) (rewriteDef)

[^10]: Rust: [`rewrite.rs:10-29`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/hir/rewrite.rs#L10-L29) (rewrite function)

[^11]: Scala: [`GraphBuilding.scala:75-120`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/GraphBuilding.scala#L75-L120) (HasSigmas, AllOf)

[^12]: Scala: [`TreeBuilding.scala:21-50`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/TreeBuilding.scala#L21-L50) (TreeBuilding trait)

[^13]: Scala: [`TreeBuilding.scala:60-100`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/TreeBuilding.scala#L60-L100) (IsArithOp, IsRelationOp)

[^14]: Scala: [`TreeBuilding.scala:100-140`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/ir/TreeBuilding.scala#L100-L140) (IsContextProperty)

[^15]: Rust: [`hir.rs:146-167`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/hir.rs#L146-L167) (ExprKind enum)

[^16]: Rust: [`hir.rs:61-94`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/hir.rs#L61-L94) (Expr::lower)
