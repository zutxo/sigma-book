# Chapter 19: Compiler Pipeline

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 16](./ch16-ergoscript-parser.md) for parsing ErgoScript to AST
- [Chapter 17](./ch17-semantic-analysis.md) for name binding and type inference
- [Chapter 18](./ch18-intermediate-representation.md) for IR optimization passes

## Learning Objectives

By the end of this chapter, you will be able to:

- Trace the complete compilation pipeline from ErgoScript source to ErgoTree bytecode
- Use the `SigmaCompiler` API to compile scripts programmatically
- Explain method call lowering strategies and when direct operations are used
- Configure compiler settings for different networks (mainnet vs testnet)

## Pipeline Architecture

The ErgoScript compiler transforms source code through multiple phases[^1][^2]:

```
Compilation Pipeline
─────────────────────────────────────────────────────

Source: "sigmaProp(SELF.value > 1000L)"
                       │
                       │ (1) Parse
                       ▼
┌─────────────────────────────────────────────────────┐
│ Untyped AST                                         │
│ Apply(Ident("sigmaProp"), [GT(Select(...), ...)])   │
└─────────────────────────────────────────────────────┘
                       │
                       │ (2) Bind
                       ▼
┌─────────────────────────────────────────────────────┐
│ Bound AST (names resolved)                          │
│ Apply(SigmaPropFunc, [GT(Self.value, 1000L)])       │
└─────────────────────────────────────────────────────┘
                       │
                       │ (3) Typecheck
                       ▼
┌─────────────────────────────────────────────────────┐
│ Typed AST                                           │
│ BoolToSigmaProp(GT(ExtractAmount(Self), 1000L))     │
│ :: SSigmaProp                                       │
└─────────────────────────────────────────────────────┘
                       │
                       │ (4) BuildGraph (Scala only)
                       ▼
┌─────────────────────────────────────────────────────┐
│ Graph IR (CSE applied)                              │
│ s1=Self, s2=s1.value, s3=1000L, s4=GT(s2,s3)        │
│ s5=sigmaProp(s4)                                    │
└─────────────────────────────────────────────────────┘
                       │
                       │ (5) BuildTree / Lower to MIR
                       ▼
┌─────────────────────────────────────────────────────┐
│ ErgoTree                                            │
│ BoolToSigmaProp(GT(ExtractAmount(Self), 1000L))     │
└─────────────────────────────────────────────────────┘
```

## Compiler Settings

Configuration controls optimization and network behavior[^3]:

```zig
const CompilerSettings = struct {
    /// Network prefix for address decoding (mainnet=0, testnet=16)
    network_prefix: u8,
    /// Whether to lower MethodCall to direct nodes
    lower_method_calls: bool,
    /// Builder for creating ErgoTree nodes
    builder: *const SigmaBuilder,

    pub fn mainnet() CompilerSettings {
        return .{
            .network_prefix = 0x00,
            .lower_method_calls = true,
            .builder = &TransformingSigmaBuilder,
        };
    }

    pub fn testnet() CompilerSettings {
        return .{
            .network_prefix = 0x10,
            .lower_method_calls = true,
            .builder = &TransformingSigmaBuilder,
        };
    }
};
```

## SigmaCompiler Implementation

The compiler orchestrates all phases[^4][^5]:

```zig
const SigmaCompiler = struct {
    settings: CompilerSettings,
    allocator: Allocator,

    pub fn init(settings: CompilerSettings, allocator: Allocator) SigmaCompiler {
        return .{
            .settings = settings,
            .allocator = allocator,
        };
    }

    /// Phase 1: Parse source to AST
    pub fn parse(self: *const SigmaCompiler, source: []const u8) !*Expr {
        var parser = Parser.init(source, self.allocator);
        return parser.parseExpr() catch |err| {
            return error.ParserError;
        };
    }

    /// Phases 2-3: Bind and typecheck
    pub fn typecheck(
        self: *const SigmaCompiler,
        env: *const ScriptEnv,
        parsed: *const Expr,
    ) !*TypedExpr {
        // Phase 2: Bind names
        const predef_registry = PredefinedFuncRegistry.init(self.settings.builder);
        var binder = Binder.init(env, self.settings.builder, self.settings.network_prefix, &predef_registry);
        const bound = try binder.bind(parsed);

        // Phase 3: Type inference and checking
        const type_env = env.collectTypes();
        var typer = Typer.init(
            self.settings.builder,
            &predef_registry,
            type_env,
            self.settings.lower_method_calls,
        );
        return typer.typecheck(bound);
    }

    /// Full compilation: all phases
    pub fn compile(
        self: *const SigmaCompiler,
        env: *const ScriptEnv,
        source: []const u8,
        ir_ctx: *IRContext,
    ) !CompilerResult {
        const parsed = try self.parse(source);
        const typed = try self.typecheck(env, parsed);
        return self.compileTyped(env, typed, ir_ctx, source);
    }

    /// Phases 4-5: Graph building and tree building
    fn compileTyped(
        self: *const SigmaCompiler,
        env: *const ScriptEnv,
        typed: *const TypedExpr,
        ir_ctx: *IRContext,
        source: []const u8,
    ) !CompilerResult {
        // Create placeholder constants for type parameters
        var placeholders_env = env.clone();
        var idx: u32 = 0;
        var iter = env.typeParams();
        while (iter.next()) |entry| {
            const placeholder = ConstantPlaceholder{
                .index = idx,
                .tpe = entry.value,
            };
            try placeholders_env.put(entry.key, .{ .placeholder = placeholder });
            idx += 1;
        }

        // Phase 4: Build graph (CSE)
        var graph_builder = GraphBuilder.init(ir_ctx, &placeholders_env);
        const compiled_graph = try graph_builder.buildGraph(typed);

        // Phase 5: Build tree (ValDef minimization)
        var tree_builder = TreeBuilder.init(ir_ctx, self.allocator);
        const compiled_tree = try tree_builder.buildTree(compiled_graph);

        return CompilerResult{
            .env = env,
            .source = source,
            .compiled_graph = compiled_graph,
            .ergo_tree = compiled_tree,
        };
    }
};

/// Result of compilation
const CompilerResult = struct {
    env: *const ScriptEnv,
    source: []const u8,
    compiled_graph: Sym,
    ergo_tree: *Expr,
};
```

## Rust Compiler Pipeline

The Rust implementation uses a direct pipeline without graph IR[^6][^7]:

```zig
/// Rust-style direct compilation pipeline
const RustCompiler = struct {
    allocator: Allocator,

    /// Compile source to ErgoTree expression
    pub fn compileExpr(
        self: *const RustCompiler,
        source: []const u8,
        env: ScriptEnv,
    ) !*MirExpr {
        // Parse to CST, then lower to HIR
        const hir = try self.compileHir(source);

        // Bind names in HIR
        var binder = Binder.init(env);
        const bound = try binder.bind(hir);

        // Assign types
        const typed = try assignType(bound);

        // Lower to MIR (ErgoTree IR)
        const mir = try lowerToMir(typed);

        // Type check MIR
        return try typeCheck(mir);
    }

    /// Compile to full ErgoTree
    pub fn compile(
        self: *const RustCompiler,
        source: []const u8,
        env: ScriptEnv,
    ) !ErgoTree {
        const expr = try self.compileExpr(source, env);
        return ErgoTree.fromExpr(expr);
    }

    fn compileHir(self: *const RustCompiler, source: []const u8) !*HirExpr {
        var parser = Parser.init(source);
        const parse_result = parser.parse();

        if (parse_result.errors.len > 0) {
            return error.ParseError;
        }

        const syntax = parse_result.syntax();
        const root = AstRoot.cast(syntax) orelse return error.InvalidRoot;
        return hirLower(root);
    }
};
```

## Method Call Lowering

Lowering transforms generic MethodCall to compact direct nodes[^8][^9]:

```
Method Call Lowering
─────────────────────────────────────────────────────

Before lowering (MethodCall - 3+ bytes):
  MethodCall(xs, CollMethods.MapMethod, [f], {})

After lowering (MapCollection - 1 byte):
  MapCollection(xs, f)

Size savings: 2+ bytes per operation
```

```zig
/// Method call lowering during typing
const MethodCallLowerer = struct {
    builder: *const SigmaBuilder,
    lower_enabled: bool,

    /// Try to lower MethodCall to direct node
    pub fn tryLower(
        self: *const MethodCallLowerer,
        obj: *const Expr,
        method: *const SMethod,
        args: []const *const Expr,
        subst: TypeSubst,
    ) ?*Expr {
        if (!self.lower_enabled) return null;

        // Check if method has IR builder
        const ir_builder = method.ir_info.ir_builder orelse return null;

        // Try to apply the builder
        return ir_builder.build(self.builder, obj, method, args, subst);
    }

    /// Unlower: convert direct nodes back to MethodCall (for display)
    pub fn unlower(self: *const MethodCallLowerer, expr: *const Expr) *Expr {
        return switch (expr.kind) {
            .multiply_group => |mg| self.builder.makeMethodCall(
                mg.left,
                &SGroupElementMethods.multiply_method,
                &[_]*const Expr{mg.right},
            ),
            .exponentiate => |exp| self.builder.makeMethodCall(
                exp.base,
                &SGroupElementMethods.exponentiate_method,
                &[_]*const Expr{exp.exponent},
            ),
            .map_collection => |mc| self.builder.makeMethodCall(
                mc.input,
                &SCollectionMethods.map_method.withConcreteTypes(.{
                    .tIV = mc.input.tpe.elemType(),
                    .tOV = mc.mapper.tpe.resultType(),
                }),
                &[_]*const Expr{mc.mapper},
            ),
            .fold => |f| self.builder.makeMethodCall(
                f.input,
                &SCollectionMethods.fold_method.withConcreteTypes(.{
                    .tIV = f.input.tpe.elemType(),
                    .tOV = f.zero.tpe,
                }),
                &[_]*const Expr{ f.zero, f.folder },
            ),
            .for_all => |fa| self.builder.makeMethodCall(
                fa.input,
                &SCollectionMethods.forall_method.withConcreteTypes(.{
                    .tIV = fa.input.tpe.elemType(),
                }),
                &[_]*const Expr{fa.predicate},
            ),
            .exists => |ex| self.builder.makeMethodCall(
                ex.input,
                &SCollectionMethods.exists_method.withConcreteTypes(.{
                    .tIV = ex.input.tpe.elemType(),
                }),
                &[_]*const Expr{ex.predicate},
            ),
            else => expr,
        };
    }
};
```

## Type Inference

Type assignment propagates and unifies types[^10][^11]:

```zig
const Typer = struct {
    builder: *const SigmaBuilder,
    predef_registry: *const PredefinedFuncRegistry,
    type_env: std.StringHashMap(SType),
    lower_method_calls: bool,

    /// Assign types to bound expression
    pub fn typecheck(self: *Typer, bound: *const Expr) !*TypedExpr {
        return self.assignType(self.type_env, bound);
    }

    fn assignType(self: *Typer, env: std.StringHashMap(SType), expr: *const Expr) !*TypedExpr {
        return switch (expr.kind) {
            .block => |b| self.typecheckBlock(env, b),
            .tuple => |t| self.typecheckTuple(env, t),
            .ident => |id| self.typecheckIdent(env, id),
            .select => |s| self.typecheckSelect(env, s),
            .apply => |a| self.typecheckApply(env, a),
            .lambda => |l| self.typecheckLambda(env, l),
            .if_expr => |i| self.typecheckIf(env, i),
            .constant => |c| self.makeTyped(c, c.tpe),
            else => error.UnsupportedExpr,
        };
    }

    fn typecheckBlock(self: *Typer, env: std.StringHashMap(SType), block: *const Block) !*TypedExpr {
        var cur_env = try env.clone();

        for (block.items) |val_def| {
            if (cur_env.contains(val_def.name)) {
                return error.DuplicateVariable;
            }
            const rhs_typed = try self.assignType(cur_env, val_def.rhs);
            try cur_env.put(val_def.name, rhs_typed.tpe);
        }

        const result_typed = try self.assignType(cur_env, block.result);
        return self.builder.makeBlock(block.items, result_typed);
    }

    fn typecheckSelect(self: *Typer, env: std.StringHashMap(SType), sel: *const Select) !*TypedExpr {
        const obj_typed = try self.assignType(env, sel.obj);

        const method = MethodsContainer.getMethod(obj_typed.tpe, sel.field) orelse
            return error.MethodNotFound;

        // Unify method receiver type with object type
        const subst = unifyTypes(method.stype.domain[0], obj_typed.tpe) orelse
            return error.TypeMismatch;

        const result_type = applySubst(method.stype.range, subst);

        // Try to lower if it's a property access (no args)
        if (self.lower_method_calls) {
            if (method.ir_info.ir_builder) |ir_builder| {
                if (ir_builder.buildProperty(self.builder, obj_typed, method)) |lowered| {
                    return lowered;
                }
            }
        }

        return self.builder.makeSelect(obj_typed, sel.field, result_type);
    }
};
```

## Error Handling

Each phase produces specific errors[^12]:

```zig
const CompileError = union(enum) {
    parse_error: ParseError,
    hir_lowering_error: HirLoweringError,
    binder_error: BinderError,
    type_error: TypeInferenceError,
    mir_lowering_error: MirLoweringError,
    type_check_error: TypeCheckError,
    ergo_tree_error: ErgoTreeError,

    pub fn prettyDesc(self: CompileError, source: []const u8) []const u8 {
        return switch (self) {
            .parse_error => |e| e.prettyDesc(source),
            .hir_lowering_error => |e| e.prettyDesc(source),
            .binder_error => |e| e.prettyDesc(source),
            .type_error => |e| e.prettyDesc(source),
            .mir_lowering_error => |e| e.prettyDesc(source),
            .type_check_error => |e| e.prettyDesc(),
            .ergo_tree_error => |e| std.fmt.allocPrint(
                allocator,
                "ErgoTree error: {any}",
                .{e},
            ) catch "format error",
        };
    }
};

/// Parse error with source location
const ParseError = struct {
    message: []const u8,
    span: TextRange,
    expected: []const TokenKind,
    found: ?TokenKind,

    pub fn prettyDesc(self: ParseError, source: []const u8) []const u8 {
        const line_info = getLineInfo(source, self.span.start);
        return std.fmt.allocPrint(allocator,
            "error: {s}\nline: {d}\n{s}\n{s}",
            .{
                self.message,
                line_info.line_num,
                line_info.line_text,
                makeUnderline(line_info, self.span),
            },
        ) catch "format error";
    }
};
```

## Predefined Functions Registry

Built-in functions are registered for name resolution[^13]:

```zig
const PredefinedFuncRegistry = struct {
    funcs: std.StringHashMap(PredefinedFunc),
    builder: *const SigmaBuilder,

    pub fn init(builder: *const SigmaBuilder) PredefinedFuncRegistry {
        var self = PredefinedFuncRegistry{
            .funcs = std.StringHashMap(PredefinedFunc).init(allocator),
            .builder = builder,
        };
        self.registerAll();
        return self;
    }

    fn registerAll(self: *PredefinedFuncRegistry) void {
        // Boolean operations
        self.register("allOf", .{
            .tpe = SFunc.init(&[_]SType{SType.collOf(.boolean)}, .boolean),
            .ir_builder = AllOfIrBuilder,
        });
        self.register("anyOf", .{
            .tpe = SFunc.init(&[_]SType{SType.collOf(.boolean)}, .boolean),
            .ir_builder = AnyOfIrBuilder,
        });

        // Sigma operations
        self.register("sigmaProp", .{
            .tpe = SFunc.init(&[_]SType{.boolean}, .sigma_prop),
            .ir_builder = SigmaPropIrBuilder,
        });
        self.register("atLeast", .{
            .tpe = SFunc.init(&[_]SType{ .int, SType.collOf(.sigma_prop) }, .sigma_prop),
            .ir_builder = AtLeastIrBuilder,
        });
        self.register("allZK", .{
            .tpe = SFunc.init(&[_]SType{SType.collOf(.sigma_prop)}, .sigma_prop),
            .ir_builder = AllZKIrBuilder,
        });
        self.register("anyZK", .{
            .tpe = SFunc.init(&[_]SType{SType.collOf(.sigma_prop)}, .sigma_prop),
            .ir_builder = AnyZKIrBuilder,
        });

        // Cryptographic
        self.register("proveDlog", .{
            .tpe = SFunc.init(&[_]SType{.group_element}, .sigma_prop),
            .ir_builder = ProveDlogIrBuilder,
        });
        self.register("proveDHTuple", .{
            .tpe = SFunc.init(&[_]SType{
                .group_element,
                .group_element,
                .group_element,
                .group_element,
            }, .sigma_prop),
            .ir_builder = ProveDHTupleIrBuilder,
        });

        // Hash functions
        self.register("blake2b256", .{
            .tpe = SFunc.init(&[_]SType{SType.collOf(.byte)}, SType.collOf(.byte)),
            .ir_builder = Blake2b256IrBuilder,
        });
        self.register("sha256", .{
            .tpe = SFunc.init(&[_]SType{SType.collOf(.byte)}, SType.collOf(.byte)),
            .ir_builder = Sha256IrBuilder,
        });

        // Global
        self.register("groupGenerator", .{
            .tpe = SFunc.init(&[_]SType{}, .group_element),
            .ir_builder = GroupGeneratorIrBuilder,
        });
    }

    fn register(self: *PredefinedFuncRegistry, name: []const u8, func: PredefinedFunc) void {
        self.funcs.put(name, func) catch unreachable;
    }
};
```

## Compilation Example

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    // Setup compiler
    const settings = CompilerSettings.testnet();
    const compiler = SigmaCompiler.init(settings, allocator);
    var ir_ctx = IRContext.init(allocator);

    // Source code
    const source =
        \\{
        \\  val deadline = 100000
        \\  val pk = PK("9fRusAarL1KkrWQVsxSRVYnvWxaAT2A96cKtNn9tvPh5XUCTgGi")
        \\  sigmaProp(HEIGHT > deadline) && pk
        \\}
    ;

    // Compile
    const env = ScriptEnv.empty();
    const result = try compiler.compile(&env, source, &ir_ctx);

    // Access results
    std.debug.print("Source: {s}\n", .{result.source});
    std.debug.print("ErgoTree: {any}\n", .{result.ergo_tree});
    std.debug.print("Type: {any}\n", .{result.ergo_tree.tpe});

    // Serialize
    const ergo_tree = try ErgoTree.fromSigmaProp(result.ergo_tree);
    const bytes = try ergo_tree.toBytes(allocator);
    std.debug.print("Bytes: {x}\n", .{std.fmt.fmtSliceHexLower(bytes)});
}
```

## Compilation Flow Detail

```
Detailed Phase Transitions
─────────────────────────────────────────────────────

Source: "OUTPUTS.exists({ (b: Box) => b.value > 100L })"

Phase 1 - Parse:
  Apply(
    Select(Ident("OUTPUTS"), "exists"),
    [Lambda(["b": Box], GT(Select(Ident("b"), "value"), 100L))]
  )

Phase 2 - Bind:
  Apply(
    Select(Context.OUTPUTS, ExistsMethod),
    [Lambda([b: SBox], GT(Select(ValUse(b), "value"), 100L))]
  )

Phase 3 - Typecheck:
  Exists(
    input: Outputs :: SColl[SBox],
    predicate: Lambda(
      args: [(0, SBox)],
      body: GT(
        ExtractAmount(ValUse(0, SBox)) :: SLong,
        LongConstant(100) :: SLong
      ) :: SBoolean
    ) :: SFunc[SBox, SBoolean]
  ) :: SBoolean

Phase 4 - BuildGraph (if using Scala IR):
  s1 = Context.OUTPUTS
  s2 = Lambda(args=[(0,SBox)], body=s3)
  s3 = GT(s4, s5)
  s4 = ValUse(0).value  // ExtractAmount
  s5 = 100L
  s6 = Exists(s1, s2)

Phase 5 - BuildTree:
  Exists(
    Outputs,
    FuncValue(
      [(1, SBox)],
      GT(ExtractAmount(ValUse(1, SBox)), LongConstant(100))
    )
  )
```

## Summary

- **5-phase pipeline**: Parse → Bind → Typecheck → BuildGraph → BuildTree
- **Method lowering** transforms MethodCall (3+ bytes) to direct nodes (1 byte)
- **Scala uses graph IR** for CSE optimization; Rust uses direct HIR→MIR
- **Type inference** propagates and unifies types through the AST
- **Predefined registry** resolves built-in function names
- **Error handling** provides detailed source-location diagnostics
- Compiler is development-time only—interpreter uses serialized ErgoTree

---

*Next: [Chapter 20: Collections](../part7/ch20-collections.md)*

[^1]: Scala: [`SigmaCompiler.scala:51-100`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/SigmaCompiler.scala#L51-L100) (SigmaCompiler class)

[^2]: Rust: [`lib.rs:16-27`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/lib.rs#L16-L27) (module structure)

[^3]: Scala: [`SigmaCompiler.scala:15-25`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/SigmaCompiler.scala#L15-L25) (CompilerSettings)

[^4]: Scala: [`SigmaCompiler.scala:55-95`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/SigmaCompiler.scala#L55-L95) (compile methods)

[^5]: Rust: [`compiler.rs:59-76`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/compiler.rs#L59-L76) (compile_expr)

[^6]: Rust: [`compiler.rs:73-76`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/compiler.rs#L73-L76) (compile)

[^7]: Rust: [`compiler.rs:78-87`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/compiler.rs#L78-L87) (compile_hir)

[^8]: Scala: [`SigmaCompiler.scala:105-150`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/SigmaCompiler.scala#L105-L150) (unlowerMethodCalls)

[^9]: Scala: [`SigmaTyper.scala:30-45`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/phases/SigmaTyper.scala#L30-L45) (processGlobalMethod)

[^10]: Scala: [`SigmaTyper.scala:50-100`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/sc/shared/src/main/scala/sigma/compiler/phases/SigmaTyper.scala#L50-L100) (assignType)

[^11]: Rust: [`type_infer.rs:25-49`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/type_infer.rs#L25-L49) (assign_type)

[^12]: Rust: [`compiler.rs:23-55`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergoscript-compiler/src/compiler.rs#L23-L55) (CompileError)

[^13]: Scala: [`SigmaPredef.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/ast/SigmaPredef.scala) (PredefinedFuncRegistry)
