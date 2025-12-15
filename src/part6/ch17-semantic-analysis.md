# Chapter 17: Semantic Analysis

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Parser ([Chapter 16](./ch16-ergoscript-parser.md))
- Type system ([Chapter 2](../part1/ch02-type-system.md))
- Basic understanding of type inference

## Learning Objectives

- Understand two-phase semantic analysis: binding then typing
- Master name resolution algorithm
- Learn type unification algorithm
- Understand method resolution and lowering
- Trace type inference for complex expressions

## Semantic Analysis Overview

After parsing, ErgoScript transforms through two phases[^1][^2]:

```
Semantic Analysis Pipeline
─────────────────────────────────────────────────────

Source Code
    │
    ▼
┌──────────────────────────────────────────────────┐
│                    PARSE                         │
│                                                  │
│  Untyped AST                                     │
│  - Identifiers have NoType                       │
│  - References are unresolved strings             │
│  - Operators are symbolic                        │
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│                    BIND                          │
│                                                  │
│  Resolve names:                                  │
│  - Global constants (HEIGHT, SELF, INPUTS)       │
│  - Environment variables                         │
│  - Predefined functions                          │
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│                    TYPE                          │
│                                                  │
│  Assign types:                                   │
│  - Infer expression types                        │
│  - Resolve method calls                          │
│  - Unify generic types                           │
│  - Check type consistency                        │
└──────────────────────────────────────────────────┘
    │
    ▼
Typed AST (ready for IR)
```

## Phase 1: Name Binding

The binder resolves identifiers to their definitions[^3][^4]:

```zig
const BinderError = struct {
    msg: []const u8,
    span: Range,

    pub fn prettyDesc(self: BinderError, source: []const u8) []const u8 {
        // Format error with source context
    }
};

const GlobalVars = enum {
    height,
    self_,
    inputs,
    outputs,
    context,
    global,
    miner_pubkey,
    last_block_utxo_root_hash,

    pub fn tpe(self: GlobalVars) SType {
        return switch (self) {
            .height => .s_int,
            .self_ => .s_box,
            .inputs => .{ .s_coll = .s_box },
            .outputs => .{ .s_coll = .s_box },
            .context => .s_context,
            .global => .s_global,
            .miner_pubkey => .{ .s_coll = .s_byte },
            .last_block_utxo_root_hash => .s_avl_tree,
        };
    }
};

const Binder = struct {
    env: ScriptEnv,
    allocator: Allocator,

    pub fn init(allocator: Allocator, env: ScriptEnv) Binder {
        return .{ .env = env, .allocator = allocator };
    }

    pub fn bind(self: *const Binder, expr: Expr) BinderError!Expr {
        return self.rewrite(expr);
    }

    fn rewrite(self: *const Binder, expr: Expr) BinderError!Expr {
        return switch (expr.kind) {
            .ident => |name| blk: {
                // Check environment first
                if (self.env.get(name)) |value| {
                    break :blk liftToConstant(value, expr.span);
                }

                // Check global variables
                if (resolveGlobal(name)) |global| {
                    break :blk .{
                        .kind = .{ .global_vars = global },
                        .span = expr.span,
                        .tpe = global.tpe(),
                    };
                }

                // Leave unresolved for typer
                break :blk expr;
            },

            .binary => |bin| blk: {
                const left = try self.rewrite(bin.lhs.*);
                const right = try self.rewrite(bin.rhs.*);
                break :blk .{
                    .kind = .{ .binary = .{
                        .op = bin.op,
                        .lhs = try self.allocator.create(Expr),
                        .rhs = try self.allocator.create(Expr),
                    } },
                    .span = expr.span,
                    .tpe = expr.tpe,
                };
            },

            .block => |block| blk: {
                var new_bindings = try self.allocator.alloc(ValDef, block.bindings.len);
                for (block.bindings, 0..) |binding, i| {
                    const rhs = try self.rewrite(binding.rhs.*);
                    new_bindings[i] = .{
                        .name = binding.name,
                        .tpe = rhs.tpe orelse binding.tpe,
                        .rhs = rhs,
                    };
                }
                const body = try self.rewrite(block.body.*);
                break :blk .{
                    .kind = .{ .block = .{
                        .bindings = new_bindings,
                        .body = body,
                    } },
                    .span = expr.span,
                    .tpe = body.tpe,
                };
            },

            .lambda => |lam| blk: {
                const body = try self.rewrite(lam.body.*);
                break :blk .{
                    .kind = .{ .lambda = .{
                        .args = lam.args,
                        .body = body,
                    } },
                    .span = expr.span,
                    .tpe = expr.tpe,
                };
            },

            else => expr,
        };
    }

    fn resolveGlobal(name: []const u8) ?GlobalVars {
        const globals = std.ComptimeStringMap(GlobalVars, .{
            .{ "HEIGHT", .height },
            .{ "SELF", .self_ },
            .{ "INPUTS", .inputs },
            .{ "OUTPUTS", .outputs },
            .{ "CONTEXT", .context },
            .{ "Global", .global },
            .{ "MinerPubkey", .miner_pubkey },
            .{ "LastBlockUtxoRootHash", .last_block_utxo_root_hash },
        });
        return globals.get(name);
    }

    fn liftToConstant(value: anytype, span: Range) Expr {
        const T = @TypeOf(value);
        return .{
            .kind = .{ .literal = switch (T) {
                i32 => .{ .int = value },
                i64 => .{ .long = value },
                bool => .{ .bool_ = value },
                else => @compileError("unsupported type"),
            } },
            .span = span,
            .tpe = SType.fromNative(T),
        };
    }
};
```

### Global Constants

```
Built-in Global Constants
─────────────────────────────────────────────────────
Name                    Type            Description
─────────────────────────────────────────────────────
HEIGHT                  Int             Current block height
SELF                    Box             Current box being spent
INPUTS                  Coll[Box]       Transaction inputs
OUTPUTS                 Coll[Box]       Transaction outputs
CONTEXT                 Context         Execution context
MinerPubkey             Coll[Byte]      Miner's public key
LastBlockUtxoRootHash   AvlTree         UTXO digest
```

## Phase 2: Type Inference

The typer assigns types to all expressions[^5][^6]:

```zig
const TyperError = struct {
    msg: []const u8,
    span: Range,
};

const TypeEnv = std.StringHashMap(SType);

const Typer = struct {
    predef_env: TypeEnv,
    lower_method_calls: bool,
    allocator: Allocator,

    pub fn init(allocator: Allocator, type_env: TypeEnv, lower: bool) Typer {
        var env = TypeEnv.init(allocator);
        // Add predefined function types
        env.put("min", .{ .s_func = .{ .args = &[_]SType{ .s_int, .s_int }, .ret = .s_int } }) catch {};
        env.put("max", .{ .s_func = .{ .args = &[_]SType{ .s_int, .s_int }, .ret = .s_int } }) catch {};
        // Merge with provided env
        var it = type_env.iterator();
        while (it.next()) |entry| {
            env.put(entry.key_ptr.*, entry.value_ptr.*) catch {};
        }
        return .{
            .predef_env = env,
            .lower_method_calls = lower,
            .allocator = allocator,
        };
    }

    pub fn typecheck(self: *Typer, bound: Expr) TyperError!Expr {
        const typed = try self.assignType(&self.predef_env, bound);
        if (typed.tpe == null) {
            return error.NoTypeAssigned;
        }
        return typed;
    }

    fn assignType(self: *Typer, env: *const TypeEnv, expr: Expr) TyperError!Expr {
        return switch (expr.kind) {
            // Identifier: lookup in environment
            .ident => |name| blk: {
                if (env.get(name)) |t| {
                    break :blk .{
                        .kind = expr.kind,
                        .span = expr.span,
                        .tpe = t,
                    };
                }
                return TyperError{
                    .msg = "Cannot assign type for variable",
                    .span = expr.span,
                };
            },

            // Global variables already typed
            .global_vars => |g| .{
                .kind = expr.kind,
                .span = expr.span,
                .tpe = g.tpe(),
            },

            // Block: extend environment with each binding
            .block => |block| blk: {
                var cur_env = env.clone();
                var new_bindings = try self.allocator.alloc(ValDef, block.bindings.len);

                for (block.bindings, 0..) |binding, i| {
                    const rhs = try self.assignType(&cur_env, binding.rhs.*);
                    try cur_env.put(binding.name, rhs.tpe.?);
                    new_bindings[i] = .{
                        .name = binding.name,
                        .tpe = rhs.tpe.?,
                        .rhs = rhs,
                    };
                }

                const body = try self.assignType(&cur_env, block.body.*);

                break :blk .{
                    .kind = .{ .block = .{
                        .bindings = new_bindings,
                        .body = body,
                    } },
                    .span = expr.span,
                    .tpe = body.tpe,
                };
            },

            // Binary: type operands, check compatibility
            .binary => |bin| blk: {
                const left = try self.assignType(env, bin.lhs.*);
                const right = try self.assignType(env, bin.rhs.*);

                const result_type = try inferBinaryType(
                    bin.op,
                    left.tpe.?,
                    right.tpe.?,
                );

                break :blk .{
                    .kind = .{ .binary = .{
                        .op = bin.op,
                        .lhs = left,
                        .rhs = right,
                    } },
                    .span = expr.span,
                    .tpe = result_type,
                };
            },

            // If: check condition is Boolean, branches have same type
            .if_ => |if_expr| blk: {
                const cond = try self.assignType(env, if_expr.cond.*);
                const then_ = try self.assignType(env, if_expr.then_.*);
                const else_ = try self.assignType(env, if_expr.else_.*);

                if (cond.tpe.? != .s_boolean) {
                    return TyperError{
                        .msg = "Condition must be Boolean",
                        .span = cond.span,
                    };
                }

                if (!typesEqual(then_.tpe.?, else_.tpe.?)) {
                    return TyperError{
                        .msg = "Branches must have same type",
                        .span = expr.span,
                    };
                }

                break :blk .{
                    .kind = .{ .if_ = .{
                        .cond = cond,
                        .then_ = then_,
                        .else_ = else_,
                    } },
                    .span = expr.span,
                    .tpe = then_.tpe,
                };
            },

            // Lambda: check argument types, type body
            .lambda => |lam| blk: {
                var lambda_env = env.clone();
                for (lam.args) |arg| {
                    if (arg.tpe == .no_type) {
                        return TyperError{
                            .msg = "Lambda argument must have explicit type",
                            .span = expr.span,
                        };
                    }
                    try lambda_env.put(arg.name, arg.tpe);
                }

                const body = try self.assignType(&lambda_env, lam.body.*);
                const func_type = SType{
                    .s_func = .{
                        .args = lam.args.map(fn(a) a.tpe),
                        .ret = body.tpe.?,
                    },
                };

                break :blk .{
                    .kind = .{ .lambda = .{
                        .args = lam.args,
                        .body = body,
                    } },
                    .span = expr.span,
                    .tpe = func_type,
                };
            },

            // Method call: type receiver, resolve method, unify types
            .select => |sel| try self.typeSelect(env, sel, expr.span),

            .apply => |app| try self.typeApply(env, app, expr.span),

            // Literals already typed
            .literal => |lit| .{
                .kind = expr.kind,
                .span = expr.span,
                .tpe = switch (lit) {
                    .int => .s_int,
                    .long => .s_long,
                    .bool_ => .s_boolean,
                    .string => .{ .s_coll = .s_byte },
                },
            },

            else => expr,
        };
    }
};
```

### Binary Operation Type Inference

```zig
fn inferBinaryType(op: BinaryOp, left: SType, right: SType) TyperError!SType {
    return switch (op) {
        // Arithmetic: operands must be same numeric type
        .plus, .minus, .multiply, .divide, .modulo => blk: {
            if (!left.isNumeric() or !right.isNumeric()) {
                return error.TypeMismatch;
            }
            if (!typesEqual(left, right)) {
                return error.TypeMismatch;
            }
            break :blk left;
        },

        // Comparison: operands must be same type, result is Boolean
        .lt, .gt, .le, .ge => blk: {
            if (!typesEqual(left, right)) {
                return error.TypeMismatch;
            }
            break :blk .s_boolean;
        },

        // Equality: operands must be same type
        .eq, .neq => blk: {
            if (!typesEqual(left, right)) {
                return error.TypeMismatch;
            }
            break :blk .s_boolean;
        },

        // Logical: Boolean operands
        .and_, .or_ => blk: {
            if (left == .s_boolean and right == .s_boolean) {
                break :blk .s_boolean;
            }
            // SigmaProp operations
            if (left == .s_sigma_prop and right == .s_sigma_prop) {
                break :blk .s_sigma_prop;
            }
            // Mixed: SigmaProp with Boolean
            if ((left == .s_sigma_prop and right == .s_boolean) or
                (left == .s_boolean and right == .s_sigma_prop))
            {
                break :blk .s_boolean;
            }
            return error.TypeMismatch;
        },

        // Bitwise: numeric operands
        .bit_and, .bit_or, .bit_xor => blk: {
            if (!left.isNumeric() or !right.isNumeric()) {
                return error.TypeMismatch;
            }
            if (!typesEqual(left, right)) {
                return error.TypeMismatch;
            }
            break :blk left;
        },
    };
}
```

## Type Unification

Finds a substitution making two types equal[^7]:

```zig
const TypeSubst = std.StringHashMap(SType);

fn unifyTypes(t1: SType, t2: SType) ?TypeSubst {
    var subst = TypeSubst.init(allocator);

    return switch (t1) {
        // Type variable matches anything
        .s_type_var => |name| blk: {
            subst.put(name, t2) catch return null;
            break :blk subst;
        },

        // Collection types: unify element types
        .s_coll => |elem1| switch (t2) {
            .s_coll => |elem2| unifyTypes(elem1, elem2),
            else => null,
        },

        // Option types: unify element types
        .s_option => |elem1| switch (t2) {
            .s_option => |elem2| unifyTypes(elem1, elem2),
            else => null,
        },

        // Tuple types: unify element-wise
        .s_tuple => |items1| switch (t2) {
            .s_tuple => |items2| blk: {
                if (items1.len != items2.len) break :blk null;
                for (items1, items2) |i1, i2| {
                    const sub = unifyTypes(i1, i2) orelse break :blk null;
                    subst = mergeSubst(subst, sub) orelse break :blk null;
                }
                break :blk subst;
            },
            else => null,
        },

        // Function types: unify domain and range
        .s_func => |f1| switch (t2) {
            .s_func => |f2| blk: {
                if (f1.args.len != f2.args.len) break :blk null;
                for (f1.args, f2.args) |a1, a2| {
                    const sub = unifyTypes(a1, a2) orelse break :blk null;
                    subst = mergeSubst(subst, sub) orelse break :blk null;
                }
                const ret_sub = unifyTypes(f1.ret, f2.ret) orelse break :blk null;
                break :blk mergeSubst(subst, ret_sub);
            },
            else => null,
        },

        // Boolean can unify with SigmaProp (implicit conversion)
        .s_boolean => switch (t2) {
            .s_sigma_prop, .s_boolean => subst,
            else => null,
        },

        // SAny matches anything
        .s_any => subst,

        // Primitive types must match exactly
        else => if (typesEqual(t1, t2)) subst else null,
    };
}

fn applySubst(tpe: SType, subst: TypeSubst) SType {
    return switch (tpe) {
        .s_type_var => |name| subst.get(name) orelse tpe,
        .s_coll => |elem| .{ .s_coll = applySubst(elem, subst) },
        .s_option => |elem| .{ .s_option = applySubst(elem, subst) },
        .s_tuple => |items| .{
            .s_tuple = items.map(fn(t) applySubst(t, subst)),
        },
        .s_func => |f| .{
            .s_func = .{
                .args = f.args.map(fn(t) applySubst(t, subst)),
                .ret = applySubst(f.ret, subst),
            },
        },
        else => tpe,
    };
}

fn mergeSubst(s1: TypeSubst, s2: TypeSubst) ?TypeSubst {
    var result = s1.clone();
    var it = s2.iterator();
    while (it.next()) |entry| {
        if (result.get(entry.key_ptr.*)) |existing| {
            if (!typesEqual(existing, entry.value_ptr.*)) {
                return null; // Conflict
            }
        } else {
            result.put(entry.key_ptr.*, entry.value_ptr.*) catch return null;
        }
    }
    return result;
}
```

### Unification Example

```
Generic Method Specialization
─────────────────────────────────────────────────────

coll.map(f) where:
  - coll: Coll[Byte]
  - map type: (Coll[T], T => R) => Coll[R]
  - f: Byte => Int

Step 1: Unify Coll[T] with Coll[Byte]
        Result: {T → Byte}

Step 2: Unify (T => R) with (Byte => Int)
        T already bound to Byte ✓
        Result: {T → Byte, R → Int}

Step 3: Apply substitution to result type
        Coll[R] → Coll[Int]

Final: map specialized to (Coll[Byte], Byte => Int) => Coll[Int]
```

## Method Resolution

Methods are looked up in type's methods container[^8]:

```zig
const MethodsContainer = struct {
    const methods_by_type = std.ComptimeStringMap([]const MethodInfo, .{
        .{ "SBox", &box_methods },
        .{ "SColl", &coll_methods },
        .{ "SContext", &context_methods },
        // ...
    });

    pub fn getMethod(tpe: SType, name: []const u8) ?MethodInfo {
        const type_name = tpe.typeName();
        if (methods_by_type.get(type_name)) |methods| {
            for (methods) |m| {
                if (std.mem.eql(u8, m.name, name)) {
                    return m;
                }
            }
        }
        return null;
    }
};

const MethodInfo = struct {
    name: []const u8,
    stype: SType,
    ir_builder: ?*const fn (Expr, []const Expr) Expr,
};

const box_methods = [_]MethodInfo{
    .{ .name = "value", .stype = .s_long, .ir_builder = null },
    .{ .name = "propositionBytes", .stype = .{ .s_coll = .s_byte }, .ir_builder = null },
    .{ .name = "id", .stype = .{ .s_coll = .s_byte }, .ir_builder = null },
    .{ .name = "tokens", .stype = .{ .s_coll = .{ .s_tuple = &[_]SType{
        .{ .s_coll = .s_byte }, .s_long,
    } } }, .ir_builder = null },
    // ...
};

const coll_methods = [_]MethodInfo{
    .{ .name = "size", .stype = .s_int, .ir_builder = &buildSizeOf },
    .{ .name = "map", .stype = .{ .s_func = .{
        .args = &[_]SType{ .{ .s_type_var = "T" }, .{ .s_func = .{
            .args = &[_]SType{.{ .s_type_var = "T" }},
            .ret = .{ .s_type_var = "R" },
        } } },
        .ret = .{ .s_coll = .{ .s_type_var = "R" } },
    } }, .ir_builder = &buildMapCollection },
    // ...
};
```

## Method Lowering

When `lower_method_calls = true`, method calls become IR nodes[^9]:

```zig
fn typeSelect(
    self: *Typer,
    env: *const TypeEnv,
    sel: SelectExpr,
    span: Range,
) TyperError!Expr {
    const receiver = try self.assignType(env, sel.obj.*);
    const receiver_type = receiver.tpe.?;

    const method = MethodsContainer.getMethod(receiver_type, sel.field) orelse {
        return TyperError{
            .msg = "Method not found",
            .span = span,
        };
    };

    // Specialize generic method type
    const specialized = specializeMethod(method.stype, receiver_type);

    // Lower to IR node if builder available
    if (method.ir_builder) |builder| {
        if (self.lower_method_calls) {
            return builder(receiver, &[_]Expr{});
        }
    }

    // Keep as method call
    return .{
        .kind = .{ .select = .{
            .obj = receiver,
            .field = sel.field,
        } },
        .span = span,
        .tpe = specialized,
    };
}

fn buildSizeOf(receiver: Expr, _: []const Expr) Expr {
    return .{
        .kind = .{ .size_of = receiver },
        .span = receiver.span,
        .tpe = .s_int,
    };
}

fn buildMapCollection(receiver: Expr, args: []const Expr) Expr {
    return .{
        .kind = .{ .map = .{
            .input = receiver,
            .mapper = args[0],
        } },
        .span = receiver.span,
        .tpe = args[0].tpe.?.s_func.ret,
    };
}
```

## MIR Lowering

After typing, HIR lowers to MIR (typed IR)[^10]:

```zig
const MirLoweringError = struct {
    msg: []const u8,
    span: Range,
};

pub fn lower(hir_expr: hir.Expr) MirLoweringError!mir.Expr {
    const mir_expr: mir.Expr = switch (hir_expr.kind) {
        .global_vars => |g| switch (g) {
            .height => mir.GlobalVars.height.toExpr(),
            .self_ => mir.GlobalVars.self_.toExpr(),
            // ...
        },

        .ident => return MirLoweringError{
            .msg = "Unresolved identifier",
            .span = hir_expr.span,
        },

        .binary => |bin| blk: {
            const left = try lower(bin.lhs.*);
            const right = try lower(bin.rhs.*);
            break :blk mir.BinOp{
                .kind = bin.op.toMirOp(),
                .left = left,
                .right = right,
            }.toExpr();
        },

        .literal => |lit| switch (lit) {
            .int => |v| mir.Constant{ .int = v }.toExpr(),
            .long => |v| mir.Constant{ .long = v }.toExpr(),
            .bool_ => |v| (if (v) mir.TrueLeaf else mir.FalseLeaf).toExpr(),
        },

        // ...
    };

    // Verify types match
    const hir_tpe = hir_expr.tpe orelse return MirLoweringError{
        .msg = "Missing type for HIR expression",
        .span = hir_expr.span,
    };

    if (!typesEqual(mir_expr.tpe(), hir_tpe)) {
        return MirLoweringError{
            .msg = "Type mismatch after lowering",
            .span = hir_expr.span,
        };
    }

    return mir_expr;
}
```

## Complete Compilation Flow

```zig
pub fn compile(source: []const u8, env: ScriptEnv) CompileError!mir.Expr {
    // 1. Parse
    const tokens = Lexer.init(source).tokenize();
    const events = Parser.init(tokens).parse();
    const ast = buildTree(events, tokens);

    // 2. Lower to HIR
    const hir = try hir.lower(ast);

    // 3. Bind
    const binder = Binder.init(allocator, env);
    const bound = try binder.bind(hir);

    // 4. Type
    const typer = Typer.init(allocator, TypeEnv.init(allocator), true);
    const typed = try typer.typecheck(bound);

    // 5. Lower to MIR
    const mir_expr = try mir.lower(typed);

    return mir_expr;
}
```

## Error Messages

```
Error Types
─────────────────────────────────────────────────────

BinderError:
  - "Variable x already defined"
  - "Cannot lift value to constant"

TyperError:
  - "Cannot assign type for variable 'foo'"
  - "Condition must be Boolean, got Int"
  - "Branches must have same type: Int vs Long"
  - "Method 'bar' not found in type Box"

MirLoweringError:
  - "Unresolved identifier"
  - "Type mismatch after lowering"
```

## Summary

Semantic analysis consists of two phases:

**Binding** (`Binder`):
- Resolves global names (HEIGHT, SELF, etc.)
- Lifts environment values to constants
- Uses bottom-up tree rewriting

**Typing** (`Typer`):
- Assigns types to all expressions
- Resolves method calls via MethodsContainer
- Unifies generic types with concrete types
- Optionally lowers method calls to IR nodes
- Checks type consistency

Key algorithms:
- **Type unification**: Find substitution making types equal
- **Substitution application**: Specialize generic types
- **Method resolution**: Look up methods in type's container

---

*Next: [Chapter 18: Intermediate Representation](./ch18-intermediate-representation.md)*

[^1]: Scala: `sc/shared/src/main/scala/sigma/compiler/phases/SigmaBinder.scala`

[^2]: Rust: `ergoscript-compiler/src/binder.rs`

[^3]: Scala: `sc/shared/src/main/scala/sigma/compiler/phases/SigmaBinder.scala:30-100`

[^4]: Rust: `ergoscript-compiler/src/binder.rs:26-61`

[^5]: Scala: `sc/shared/src/main/scala/sigma/compiler/phases/SigmaTyper.scala`

[^6]: Rust: `ergoscript-compiler/src/type_infer.rs`

[^7]: Scala: `core/shared/src/main/scala/sigma/ast/package.scala` (unifyTypes)

[^8]: Scala: `data/shared/src/main/scala/sigma/reflection/SRMethod.scala`

[^9]: Scala: `sc/shared/src/main/scala/sigma/compiler/phases/SigmaTyper.scala:200-280`

[^10]: Rust: `ergoscript-compiler/src/mir/lower.rs:29-76`