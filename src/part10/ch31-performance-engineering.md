# Chapter 31: Performance Engineering

## Prerequisites

- Evaluation model ([Chapter 12](../part5/ch12-evaluation-model.md))
- Serialization ([Chapter 7](../part2/ch07-serialization.md))
- Cost model ([Chapter 13](../part5/ch13-cost-model.md))

## Learning Objectives

- Understand performance-critical patterns in interpreters
- Master comptime for zero-cost abstractions
- Apply data-oriented design for cache efficiency
- Use SIMD and vectorization for throughput
- Profile and benchmark systematically

## Performance Architecture

Script interpretation requires processing thousands of transactions per block[^1][^2]:

```
Performance Critical Paths
══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    Transaction Flow                              │
│                                                                  │
│   Block (1000+ txs)                                             │
│       │                                                         │
│       ├── Tx 1: 3 inputs × (deserialize + evaluate + verify)   │
│       ├── Tx 2: 1 input × (deserialize + evaluate + verify)    │
│       ├── Tx 3: 5 inputs × (deserialize + evaluate + verify)   │
│       └── ...                                                   │
│                                                                  │
│   Hot paths (per input):                                        │
│     • Deserialization: ~50-200 opcode parses                   │
│     • Evaluation: ~100-500 operations                          │
│     • Proof verification: 1-10 EC operations                   │
└─────────────────────────────────────────────────────────────────┘

Performance Targets:
  Deserialization:   < 100 µs per script
  Evaluation:        < 500 µs per script
  Verification:      < 2 ms per input
  Total per block:   < 5 seconds
```

## Comptime Optimization

Zig's comptime enables zero-cost abstractions:

```zig
/// Compile-time type dispatch eliminates runtime branching
fn evalOperation(comptime op: OpCode, args: []const Value) !Value {
    return switch (op) {
        .plus => evalPlus(args),
        .minus => evalMinus(args),
        .multiply => evalMultiply(args),
        // All branches resolved at compile time
        inline else => |o| evalGeneric(o, args),
    };
}

/// Comptime-generated lookup tables
const op_costs = blk: {
    var costs: [256]u32 = undefined;
    for (0..256) |i| {
        costs[i] = computeCost(@enumFromInt(i));
    }
    break :blk costs;
};

/// Zero-cost field access via comptime offset calculation
fn getField(comptime T: type, comptime field: []const u8, ptr: *const T) *const @TypeOf(@field(T{}, field)) {
    const offset = @offsetOf(T, field);
    const byte_ptr: [*]const u8 = @ptrCast(ptr);
    return @ptrCast(@alignCast(byte_ptr + offset));
}
```

## Data-Oriented Design

Structure data for cache efficiency:

```zig
/// Bad: Array of Structs (AoS) - poor cache locality for iteration
const ValueAoS = struct {
    tag: ValueTag,      // 1 byte
    padding: [7]u8,     // 7 bytes padding
    data: [8]u8,        // 8 bytes payload
}; // 16 bytes per value, only 9 used

/// Good: Struct of Arrays (SoA) - excellent cache locality
const ValueStore = struct {
    tags: []ValueTag,           // Packed tags
    data: [][8]u8,              // Packed payloads
    len: usize,

    /// Iterate tags without touching payload
    pub fn countType(self: *const ValueStore, target: ValueTag) usize {
        var count: usize = 0;
        for (self.tags) |tag| {
            count += @intFromBool(tag == target);
        }
        return count;
    }

    /// Access specific value
    pub fn get(self: *const ValueStore, idx: usize) Value {
        return Value.decode(self.tags[idx], self.data[idx]);
    }
};
```

### Memory Layout Analysis

```
Cache Line Utilization
══════════════════════════════════════════════════════════════════

Array of Structs (AoS):
┌──────────────────────────────────────────────────────────────────┐
│ Cache Line (64 bytes)                                            │
├──────────────────────────────────────────────────────────────────┤
│ Value[0] │ Value[1] │ Value[2] │ Value[3] │                      │
│ 16 bytes │ 16 bytes │ 16 bytes │ 16 bytes │                      │
│ T+P+D    │ T+P+D    │ T+P+D    │ T+P+D    │ Wasted               │
└──────────────────────────────────────────────────────────────────┘
Tag iteration: 25% cache utilization (touches only 1 byte per 16)

Struct of Arrays (SoA):
┌──────────────────────────────────────────────────────────────────┐
│ Tags Cache Line (64 bytes)                                       │
├──────────────────────────────────────────────────────────────────┤
│ T[0] T[1] T[2] ... T[63]                                        │
│ 64 tags in single cache line                                    │
└──────────────────────────────────────────────────────────────────┘
Tag iteration: 100% cache utilization (64 values per fetch)

Speedup: ~4x for tag-only operations
```

## Arena Allocators

Batch allocations reduce overhead:

```zig
const ArenaAllocator = std.heap.ArenaAllocator;

/// Evaluation context with arena for temporary allocations
const EvalContext = struct {
    arena: ArenaAllocator,
    constants: []const Constant,
    env: Environment,

    pub fn init(backing: Allocator) EvalContext {
        return .{
            .arena = ArenaAllocator.init(backing),
            .constants = &[_]Constant{},
            .env = Environment.init(),
        };
    }

    /// All temporary allocations use arena
    pub fn allocTemp(self: *EvalContext, comptime T: type, n: usize) ![]T {
        return self.arena.allocator().alloc(T, n);
    }

    /// Single deallocation frees all temps
    pub fn reset(self: *EvalContext) void {
        _ = self.arena.reset(.retain_capacity);
    }

    pub fn deinit(self: *EvalContext) void {
        self.arena.deinit();
    }
};

/// Usage: batch evaluation without per-operation allocations
fn evaluateScript(tree: *const ErgoTree, allocator: Allocator) !Value {
    var ctx = EvalContext.init(allocator);
    defer ctx.deinit();

    for (tree.ops) |op| {
        try evalOp(op, &ctx);
    }

    ctx.reset(); // Free all temps at once
    return ctx.result;
}
```

## Loop Optimization

Efficient iteration patterns:

```zig
/// Unrolled loop for fixed-size operations
fn hashBlock(data: *const [64]u8, state: *[8]u32) void {
    // Process 16 words per iteration, unrolled
    comptime var i: usize = 0;
    inline while (i < 64) : (i += 4) {
        const w0 = std.mem.readInt(u32, data[i..][0..4], .big);
        const w1 = std.mem.readInt(u32, data[i + 4..][0..4], .big);
        const w2 = std.mem.readInt(u32, data[i + 8..][0..4], .big);
        const w3 = std.mem.readInt(u32, data[i + 12..][0..4], .big);
        round(state, w0);
        round(state, w1);
        round(state, w2);
        round(state, w3);
    }
}

/// Vectorized collection operations
fn sumValues(values: []const i64) i64 {
    const Vec = @Vector(4, i64);
    var sum_vec: Vec = @splat(0);

    var i: usize = 0;
    while (i + 4 <= values.len) : (i += 4) {
        const chunk: Vec = values[i..][0..4].*;
        sum_vec += chunk;
    }

    // Reduce vector to scalar
    var sum = @reduce(.Add, sum_vec);

    // Handle remainder
    while (i < values.len) : (i += 1) {
        sum += values[i];
    }

    return sum;
}
```

## Memoization

Cache expensive computations:

```zig
/// Generic memoization with comptime key type
fn Memoized(comptime K: type, comptime V: type) type {
    return struct {
        cache: std.AutoHashMap(K, V),

        const Self = @This();

        pub fn init(allocator: Allocator) Self {
            return .{ .cache = std.AutoHashMap(K, V).init(allocator) };
        }

        pub fn getOrCompute(
            self: *Self,
            key: K,
            compute: *const fn (K) V,
        ) V {
            const result = self.cache.getOrPut(key) catch unreachable;
            if (!result.found_existing) {
                result.value_ptr.* = compute(key);
            }
            return result.value_ptr.*;
        }

        pub fn reset(self: *Self) void {
            self.cache.clearRetainingCapacity();
        }
    };
}

/// Type method resolution memoization
const MethodCache = Memoized(struct { type_code: u8, method_id: u8 }, *const Method);

var method_cache: MethodCache = undefined;

fn resolveMethod(type_code: u8, method_id: u8) *const Method {
    return method_cache.getOrCompute(
        .{ .type_code = type_code, .method_id = method_id },
        computeMethod,
    );
}
```

## String Interning

Avoid repeated string allocations:

```zig
const StringInterner = struct {
    table: std.StringHashMap([]const u8),
    arena: ArenaAllocator,

    pub fn init(allocator: Allocator) StringInterner {
        return .{
            .table = std.StringHashMap([]const u8).init(allocator),
            .arena = ArenaAllocator.init(allocator),
        };
    }

    /// Return interned string (pointer stable for lifetime)
    pub fn intern(self: *StringInterner, str: []const u8) []const u8 {
        if (self.table.get(str)) |existing| {
            return existing;
        }

        // Allocate permanent copy
        const copy = self.arena.allocator().dupe(u8, str) catch unreachable;
        self.table.put(copy, copy) catch unreachable;
        return copy;
    }
};

// Variable names are interned for fast comparison
fn lookupVar(env: *const Environment, name: []const u8) ?Value {
    const interned = global_interner.intern(name);
    return env.bindings.get(interned);
}
```

## SIMD for Crypto

Vectorized elliptic curve operations:

```zig
/// SIMD-accelerated field multiplication (mod p)
fn mulModP(a: *const [4]u64, b: *const [4]u64) [4]u64 {
    // Use vector operations where available
    if (comptime std.Target.current.cpu.arch.isX86()) {
        return mulModP_avx2(a, b);
    } else if (comptime std.Target.current.cpu.arch.isAARCH64()) {
        return mulModP_neon(a, b);
    } else {
        return mulModP_scalar(a, b);
    }
}

fn mulModP_avx2(a: *const [4]u64, b: *const [4]u64) [4]u64 {
    // AVX2 implementation using 256-bit vectors
    const va: @Vector(4, u64) = a.*;
    const vb: @Vector(4, u64) = b.*;

    // Schoolbook multiplication with vector operations
    // ... (optimized implementation)

    return result;
}
```

## Profiling and Benchmarking

Built-in profiling support:

```zig
const Timer = struct {
    start: i128,

    pub fn init() Timer {
        return .{ .start = std.time.nanoTimestamp() };
    }

    pub fn elapsed(self: *const Timer) u64 {
        const now = std.time.nanoTimestamp();
        return @intCast(now - self.start);
    }
};

/// Benchmark harness
fn benchmark(
    comptime name: []const u8,
    comptime iterations: usize,
    comptime warmup: usize,
    func: *const fn () void,
) void {
    // Warmup
    for (0..warmup) |_| {
        func();
    }

    // Measure
    const timer = Timer.init();
    for (0..iterations) |_| {
        func();
    }
    const total_ns = timer.elapsed();

    const ns_per_op = total_ns / iterations;
    const ops_per_sec = @as(f64, 1_000_000_000) / @as(f64, @floatFromInt(ns_per_op));

    std.debug.print("{s}: {} ns/op ({d:.0} ops/sec)\n", .{
        name,
        ns_per_op,
        ops_per_sec,
    });
}

// Usage
test "benchmark deserialization" {
    benchmark("deserialize_script", 10000, 1000, struct {
        fn run() void {
            _ = deserialize(test_script);
        }
    }.run);
}
```

## Memory Profiling

Track allocations in debug builds:

```zig
const DebugAllocator = struct {
    backing: Allocator,
    total_allocated: usize = 0,
    total_freed: usize = 0,
    allocation_count: usize = 0,

    pub fn allocator(self: *DebugAllocator) Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *DebugAllocator = @ptrCast(@alignCast(ctx));
        self.total_allocated += len;
        self.allocation_count += 1;
        return self.backing.rawAlloc(len, ptr_align, ret_addr);
    }

    // ... other methods

    pub fn report(self: *const DebugAllocator) void {
        std.debug.print("Allocations: {}\n", .{self.allocation_count});
        std.debug.print("Total allocated: {} bytes\n", .{self.total_allocated});
        std.debug.print("Total freed: {} bytes\n", .{self.total_freed});
        std.debug.print("Leaked: {} bytes\n", .{self.total_allocated - self.total_freed});
    }
};
```

## Performance Patterns

```
Optimization Decision Tree
══════════════════════════════════════════════════════════════════

Is operation in hot path?
│
├── NO → Optimize for clarity, not speed
│
└── YES → Profile first, then:
    │
    ├── CPU-bound?
    │   ├── Use comptime for dispatch
    │   ├── Unroll small loops
    │   ├── Use SIMD where applicable
    │   └── Inline critical functions
    │
    ├── Memory-bound?
    │   ├── Use SoA layout
    │   ├── Pool/arena allocators
    │   ├── Reduce allocations
    │   └── Prefetch data
    │
    └── Allocation-bound?
        ├── Arena allocators
        ├── Object pools
        ├── String interning
        └── Stack allocation where safe
```

## Performance Checklist

When writing performance-critical code:

```zig
// ✓ Use comptime for type-level decisions
const Handler = comptime getHandler(op);

// ✓ Pre-compute lookup tables
const costs = comptime computeCostTable();

// ✓ Use SoA for iterated data
const Store = struct { tags: []Tag, values: []Value };

// ✓ Arena allocators for batch operations
var arena = ArenaAllocator.init(allocator);
defer arena.deinit();

// ✓ Inline hot functions
pub inline fn addCost(self: *CostAccum, cost: u32) !void

// ✓ Avoid allocations in tight loops
for (items) |item| {
    // Process without allocation
}

// ✓ Use vectors for parallel data
const Vec4 = @Vector(4, u64);

// ✓ Profile before optimizing
// std.debug.print("elapsed: {} ns\n", .{timer.elapsed()});
```

## Summary

- **Comptime** enables zero-cost abstractions and compile-time dispatch
- **Data-oriented design** (SoA) improves cache efficiency 4x+
- **Arena allocators** batch deallocations for throughput
- **Loop unrolling** and SIMD accelerate hot paths
- **Memoization** caches expensive computations
- **String interning** reduces allocation pressure
- **Profile first** before optimizing—measure, don't guess

---

*Next: [Appendix A: Type Codes](../appendices/appendix-a-type-codes.md)*

[^1]: Scala: `docs/perf-style-guide.md` (HOTSPOT patterns)

[^2]: Rust: Performance-oriented design throughout sigma-rust crates

[^3]: Scala: `core/shared/src/main/scala/sigma/util/MemoizedFunc.scala`

[^4]: Rust: Memoization patterns in ergotree-interpreter

[^5]: Scala: `interpreter/shared/src/main/scala/sigmastate/eval/CErgoTreeEvaluator.scala` (fixedCostOp)

[^6]: Rust: `ergotree-interpreter/src/eval.rs` (cost tracking)
