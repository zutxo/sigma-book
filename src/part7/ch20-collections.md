# Chapter 20: Collections

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 2](../part1/ch02-type-system.md) for `Coll[T]` type and type parameters
- [Chapter 5](../part2/ch05-operations-opcodes.md) for collection operation opcodes
- [Chapter 12](../part5/ch12-evaluation-model.md) for how collection operations are evaluated

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the `Coll[T]` interface and its core operations (map, filter, fold, etc.)
- Implement array-backed collections with bounds checking
- Describe the Structure-of-Arrays optimization for pair collections
- Use `CollBuilder` for creating and manipulating collections
- Understand cost implications of collection operations

## Collection Architecture

Collections in ErgoScript are immutable, indexed sequences[^1][^2]:

```
Collection Architecture
─────────────────────────────────────────────────────

                    Coll[T]
                       │
         ┌─────────────┴─────────────┐
         │                           │
   CollOverArray[T]            PairColl[L,R]
   (standard array)         (structure-of-arrays)
         │                           │
         │                    ┌──────┴──────┐
    Array[T]                Coll[L]      Coll[R]
                           (left)       (right)
```

## Coll[T] Interface

Core collection interface with specialized operations[^3][^4]:

```zig
/// Immutable indexed collection
const Coll = struct {
    data: CollData,
    elem_type: SType,
    builder: *CollBuilder,

    const CollData = union(enum) {
        /// Standard array-backed collection
        array: ArrayColl,
        /// Optimized pair collection
        pair: PairCollData,
    };

    /// Number of elements
    pub fn length(self: *const Coll) usize {
        return switch (self.data) {
            .array => |a| a.items.len,
            .pair => |p| @min(p.ls.length(), p.rs.length()),
        };
    }

    pub fn size(self: *const Coll) usize {
        return self.length();
    }

    pub fn isEmpty(self: *const Coll) bool {
        return self.length() == 0;
    }

    /// Element at index (0-based)
    pub fn get(self: *const Coll, i: usize) ?Value {
        if (i >= self.length()) return null;
        return switch (self.data) {
            .array => |a| a.items[i],
            .pair => |p| Value.tuple(.{ p.ls.get(i).?, p.rs.get(i).? }),
        };
    }

    /// Element at index with default
    pub fn getOrElse(self: *const Coll, i: usize, default: Value) Value {
        return self.get(i) orelse default;
    }

    /// Element access (throws on out of bounds)
    pub fn apply(self: *const Coll, i: usize) !Value {
        return self.get(i) orelse error.IndexOutOfBounds;
    }
};
```

## Transformation Operations

Map, filter, and fold with cost tracking[^5][^6]:

```zig
/// Collection transformation operations
const CollTransforms = struct {

    /// Apply function to each element
    pub fn map(
        coll: *const Coll,
        mapper: *const Closure,
        E: *Evaluator,
    ) !*Coll {
        const n = coll.length();
        try E.addSeqCost(MapCost, n, OpCode.Map);

        var result = try E.allocator.alloc(Value, n);
        for (0..n) |i| {
            const elem = coll.get(i).?;
            try E.addCost(AddToEnvCost, OpCode.Map);
            var fn_env = try mapper.captured_env.extend(mapper.args[0].id, elem);
            result[i] = try mapper.body.eval(&fn_env, E);
        }

        return coll.builder.fromArray(result, mapper.result_type);
    }

    /// Select elements satisfying predicate
    pub fn filter(
        coll: *const Coll,
        predicate: *const Closure,
        E: *Evaluator,
    ) !*Coll {
        const n = coll.length();
        try E.addSeqCost(FilterCost, n, OpCode.Filter);

        var result = std.ArrayList(Value).init(E.allocator);
        for (0..n) |i| {
            const elem = coll.get(i).?;
            try E.addCost(AddToEnvCost, OpCode.Filter);
            var fn_env = try predicate.captured_env.extend(predicate.args[0].id, elem);
            const keep = try E.evalTo(bool, &fn_env, predicate.body);
            if (keep) {
                try result.append(elem);
            }
        }

        return coll.builder.fromArray(result.items, coll.elem_type);
    }

    /// Left-associative fold
    pub fn foldLeft(
        coll: *const Coll,
        zero: Value,
        folder: *const Closure,
        E: *Evaluator,
    ) !Value {
        const n = coll.length();
        try E.addSeqCost(FoldCost, n, OpCode.Fold);

        var accum = zero;
        for (0..n) |i| {
            const elem = coll.get(i).?;
            const tuple = Value.tuple(.{ accum, elem });
            try E.addCost(AddToEnvCost, OpCode.Fold);
            var fn_env = try folder.captured_env.extend(folder.args[0].id, tuple);
            accum = try folder.body.eval(&fn_env, E);
        }

        return accum;
    }

    /// Flatten nested collections
    pub fn flatMap(
        coll: *const Coll,
        mapper: *const Closure,
        E: *Evaluator,
    ) !*Coll {
        const n = coll.length();
        var result = std.ArrayList(Value).init(E.allocator);

        for (0..n) |i| {
            const elem = coll.get(i).?;
            try E.addCost(AddToEnvCost, OpCode.FlatMap);
            var fn_env = try mapper.captured_env.extend(mapper.args[0].id, elem);
            const inner_coll = try E.evalTo(*Coll, &fn_env, mapper.body);

            for (0..inner_coll.length()) |j| {
                try result.append(inner_coll.get(j).?);
            }
        }

        return coll.builder.fromArray(result.items, mapper.result_type);
    }
};

const MapCost = PerItemCost{
    .base = JitCost{ .value = 10 },
    .per_chunk = JitCost{ .value = 5 },
    .chunk_size = 10,
};

const FilterCost = PerItemCost{
    .base = JitCost{ .value = 20 },
    .per_chunk = JitCost{ .value = 5 },
    .chunk_size = 10,
};

const FoldCost = PerItemCost{
    .base = JitCost{ .value = 10 },
    .per_chunk = JitCost{ .value = 5 },
    .chunk_size = 10,
};
```

## Predicate Operations

Exists, forall with short-circuit evaluation[^7]. Note: Short-circuit behavior means execution time varies based on collection contents. This is acceptable in blockchain contexts where data is public, but would be a timing side-channel if collections contained secrets.

```zig
/// Predicate operations (short-circuit)
const CollPredicates = struct {

    /// True if any element satisfies predicate
    pub fn exists(
        coll: *const Coll,
        predicate: *const Closure,
        E: *Evaluator,
    ) !bool {
        const n = coll.length();

        for (0..n) |i| {
            const elem = coll.get(i).?;
            try E.addCost(AddToEnvCost, OpCode.Exists);
            var fn_env = try predicate.captured_env.extend(predicate.args[0].id, elem);
            const result = try E.evalTo(bool, &fn_env, predicate.body);

            if (result) {
                // Short-circuit: found matching element
                try E.addSeqCost(ExistsCost, i + 1, OpCode.Exists);
                return true;
            }
        }

        try E.addSeqCost(ExistsCost, n, OpCode.Exists);
        return false;
    }

    /// True if all elements satisfy predicate
    pub fn forall(
        coll: *const Coll,
        predicate: *const Closure,
        E: *Evaluator,
    ) !bool {
        const n = coll.length();

        for (0..n) |i| {
            const elem = coll.get(i).?;
            try E.addCost(AddToEnvCost, OpCode.ForAll);
            var fn_env = try predicate.captured_env.extend(predicate.args[0].id, elem);
            const result = try E.evalTo(bool, &fn_env, predicate.body);

            if (!result) {
                // Short-circuit: found non-matching element
                try E.addSeqCost(ForAllCost, i + 1, OpCode.ForAll);
                return false;
            }
        }

        try E.addSeqCost(ForAllCost, n, OpCode.ForAll);
        return true;
    }

    /// Find first element satisfying predicate
    pub fn find(
        coll: *const Coll,
        predicate: *const Closure,
        E: *Evaluator,
    ) !?Value {
        const n = coll.length();

        for (0..n) |i| {
            const elem = coll.get(i).?;
            var fn_env = try predicate.captured_env.extend(predicate.args[0].id, elem);
            const result = try E.evalTo(bool, &fn_env, predicate.body);

            if (result) {
                return elem;
            }
        }

        return null;
    }

    /// Index of first element satisfying predicate
    pub fn indexWhere(
        coll: *const Coll,
        predicate: *const Closure,
        from: usize,
        E: *Evaluator,
    ) !i32 {
        const n = coll.length();
        const start = @max(from, 0);

        for (start..n) |i| {
            const elem = coll.get(i).?;
            var fn_env = try predicate.captured_env.extend(predicate.args[0].id, elem);
            const result = try E.evalTo(bool, &fn_env, predicate.body);

            if (result) {
                return @intCast(i);
            }
        }

        return -1;  // Not found
    }
};
```

## Slicing Operations

Slice, take, append[^8]:

```zig
/// Slicing operations
const CollSlicing = struct {

    /// First n elements
    pub fn take(coll: *const Coll, n: usize, E: *Evaluator) !*Coll {
        if (n <= 0) return coll.builder.emptyColl(coll.elem_type);
        if (n >= coll.length()) return coll;

        try E.addSeqCost(SliceCost, n, OpCode.Slice);
        return coll.builder.fromSlice(coll, 0, n);
    }

    /// Elements from index `from` until `until`
    pub fn slice(
        coll: *const Coll,
        from: usize,
        until: usize,
        E: *Evaluator,
    ) !*Coll {
        const actual_from = @min(from, coll.length());
        const actual_until = @min(until, coll.length());
        const len = if (actual_until > actual_from) actual_until - actual_from else 0;

        try E.addSeqCost(SliceCost, len, OpCode.Slice);
        return coll.builder.fromSlice(coll, actual_from, actual_until);
    }

    /// Concatenate collections
    pub fn append(coll: *const Coll, other: *const Coll, E: *Evaluator) !*Coll {
        if (coll.length() == 0) return other;
        if (other.length() == 0) return coll;

        const total = coll.length() + other.length();
        try E.addSeqCost(AppendCost, total, OpCode.Append);

        var result = try E.allocator.alloc(Value, total);
        for (0..coll.length()) |i| {
            result[i] = coll.get(i).?;
        }
        for (0..other.length()) |i| {
            result[coll.length() + i] = other.get(i).?;
        }

        return coll.builder.fromArray(result, coll.elem_type);
    }

    /// Replace slice with patch
    pub fn patch(
        coll: *const Coll,
        from: usize,
        replacement: *const Coll,
        replaced: usize,
        E: *Evaluator,
    ) !*Coll {
        const before = coll.slice(0, from, E);
        const after = coll.slice(from + replaced, coll.length(), E);
        const temp = try before.append(replacement, E);
        return temp.append(after, E);
    }

    /// Replace single element
    pub fn updated(
        coll: *const Coll,
        index: usize,
        elem: Value,
        E: *Evaluator,
    ) !*Coll {
        if (index >= coll.length()) return error.IndexOutOfBounds;

        var result = try E.allocator.alloc(Value, coll.length());
        for (0..coll.length()) |i| {
            result[i] = if (i == index) elem else coll.get(i).?;
        }

        return coll.builder.fromArray(result, coll.elem_type);
    }
};
```

## Structure-of-Arrays: PairColl

Optimized representation for collections of pairs[^9][^10]:

```
Structure-of-Arrays vs Array-of-Structures
─────────────────────────────────────────────────────

Array-of-Structures (standard):
┌────────────────────────────────────────────────────┐
│ [(L0,R0), (L1,R1), (L2,R2), (L3,R3), (L4,R4)]      │
│                                                    │
│ Memory: L0 R0 L1 R1 L2 R2 L3 R3 L4 R4              │
│ Issue: Cache unfriendly when accessing only Ls     │
└────────────────────────────────────────────────────┘

Structure-of-Arrays (PairColl):
┌────────────────────────────────────────────────────┐
│ ls: [L0, L1, L2, L3, L4]                           │
│ rs: [R0, R1, R2, R3, R4]                           │
│                                                    │
│ Memory: L0 L1 L2 L3 L4 | R0 R1 R2 R3 R4            │
│ Benefit: Cache friendly, O(1) unzip                │
└────────────────────────────────────────────────────┘
```

```zig
/// Optimized pair collection (Structure-of-Arrays)
const PairColl = struct {
    ls: *Coll,  // Left components
    rs: *Coll,  // Right components
    builder: *CollBuilder,

    pub fn length(self: *const PairColl) usize {
        return @min(self.ls.length(), self.rs.length());
    }

    /// Element at index returns tuple
    pub fn get(self: *const PairColl, i: usize) ?Value {
        const l = self.ls.get(i) orelse return null;
        const r = self.rs.get(i) orelse return null;
        return Value.tuple(.{ l, r });
    }

    /// O(1) unzip - just return components
    pub fn unzip(self: *const PairColl) struct { *Coll, *Coll } {
        return .{ self.ls, self.rs };
    }

    /// Map only left components
    pub fn mapFirst(
        self: *const PairColl,
        mapper: *const Closure,
        E: *Evaluator,
    ) !*PairColl {
        const mapped_ls = try CollTransforms.map(self.ls, mapper, E);
        return self.builder.pairColl(mapped_ls, self.rs);
    }

    /// Map only right components
    pub fn mapSecond(
        self: *const PairColl,
        mapper: *const Closure,
        E: *Evaluator,
    ) !*PairColl {
        const mapped_rs = try CollTransforms.map(self.rs, mapper, E);
        return self.builder.pairColl(self.ls, mapped_rs);
    }

    /// Slice maintains structure-of-arrays
    pub fn slice(
        self: *const PairColl,
        from: usize,
        until: usize,
        E: *Evaluator,
    ) !*PairColl {
        const sliced_ls = try CollSlicing.slice(self.ls, from, until, E);
        const sliced_rs = try CollSlicing.slice(self.rs, from, until, E);
        return self.builder.pairColl(sliced_ls, sliced_rs);
    }

    /// Append maintains structure
    pub fn append(
        self: *const PairColl,
        other: *const PairColl,
        E: *Evaluator,
    ) !*PairColl {
        const combined_ls = try CollSlicing.append(self.ls, other.ls, E);
        const combined_rs = try CollSlicing.append(self.rs, other.rs, E);
        return self.builder.pairColl(combined_ls, combined_rs);
    }
};
```

## CollBuilder

Factory for creating collections[^11][^12]:

```zig
/// Factory for creating collections
const CollBuilder = struct {
    allocator: Allocator,

    /// Create pair collection from two collections
    pub fn pairColl(
        self: *CollBuilder,
        ls: *Coll,
        rs: *Coll,
    ) *PairColl {
        // Handle length mismatch by using minimum
        const result = self.allocator.create(PairColl) catch unreachable;
        result.* = .{
            .ls = ls,
            .rs = rs,
            .builder = self,
        };
        return result;
    }

    /// Create collection from array
    pub fn fromArray(
        self: *CollBuilder,
        items: []const Value,
        elem_type: SType,
    ) *Coll {
        // Enforce size limit
        if (items.len > MAX_ARRAY_LENGTH) {
            @panic("Collection size exceeds maximum");
        }

        const result = self.allocator.create(Coll) catch unreachable;

        // Special handling for pairs → PairColl
        if (elem_type == .s_tuple and elem_type.s_tuple.items.len == 2) {
            const ls = self.allocator.alloc(Value, items.len) catch unreachable;
            const rs = self.allocator.alloc(Value, items.len) catch unreachable;
            for (items, 0..) |item, i| {
                ls[i] = item.tuple[0];
                rs[i] = item.tuple[1];
            }
            result.* = .{
                .data = .{ .pair = .{
                    .ls = self.fromArray(ls, elem_type.s_tuple.items[0]),
                    .rs = self.fromArray(rs, elem_type.s_tuple.items[1]),
                } },
                .elem_type = elem_type,
                .builder = self,
            };
        } else {
            result.* = .{
                .data = .{ .array = .{ .items = items } },
                .elem_type = elem_type,
                .builder = self,
            };
        }
        return result;
    }

    /// Create collection of n copies of value
    pub fn replicate(
        self: *CollBuilder,
        n: usize,
        value: Value,
        elem_type: SType,
    ) *Coll {
        var items = self.allocator.alloc(Value, n) catch unreachable;
        for (items) |*item| {
            item.* = value;
        }
        return self.fromArray(items, elem_type);
    }

    /// Create empty collection
    pub fn emptyColl(self: *CollBuilder, elem_type: SType) *Coll {
        return self.fromArray(&[_]Value{}, elem_type);
    }

    /// Split pair collection into two collections
    pub fn unzip(self: *CollBuilder, coll: *const Coll) struct { *Coll, *Coll } {
        switch (coll.data) {
            .pair => |p| {
                // O(1) for PairColl
                return .{ p.ls, p.rs };
            },
            .array => |a| {
                // O(n) for regular collection - must materialize
                const n = a.items.len;
                var ls = self.allocator.alloc(Value, n) catch unreachable;
                var rs = self.allocator.alloc(Value, n) catch unreachable;
                for (a.items, 0..) |item, i| {
                    ls[i] = item.tuple[0];
                    rs[i] = item.tuple[1];
                }
                const elem_type = coll.elem_type.s_tuple;
                return .{
                    self.fromArray(ls, elem_type.items[0]),
                    self.fromArray(rs, elem_type.items[1]),
                };
            },
        }
    }

    /// Element-wise XOR of byte arrays
    pub fn xor(self: *CollBuilder, left: *const Coll, right: *const Coll) *Coll {
        const n = @min(left.length(), right.length());
        var result = self.allocator.alloc(Value, n) catch unreachable;
        for (0..n) |i| {
            const l = left.get(i).?.byte;
            const r = right.get(i).?.byte;
            result[i] = Value{ .byte = l ^ r };
        }
        return self.fromArray(result, .byte);
    }
};

/// Maximum collection size (DoS protection)
const MAX_ARRAY_LENGTH: usize = 100_000;
```

## Rust Collection Representation

The Rust implementation uses a different approach[^13][^14]:

```zig
/// Rust-style Collection enum
const RustCollection = union(enum) {
    /// Special representation for boolean constants (bit-packed)
    bool_constants: []const bool,
    /// Collection of expressions
    exprs: struct {
        elem_type: SType,
        items: []const *Expr,
    },

    pub fn tpe(self: RustCollection) SType {
        return switch (self) {
            .bool_constants => SType.collOf(.boolean),
            .exprs => |e| SType.collOf(e.elem_type),
        };
    }

    pub fn opCode(self: RustCollection) OpCode {
        return switch (self) {
            .bool_constants => OpCode.CollOfBoolConst,
            .exprs => OpCode.Coll,
        };
    }
};

/// Rust collection serialization
fn serializeCollection(coll: RustCollection, writer: anytype) !void {
    switch (coll) {
        .bool_constants => |bools| {
            try writer.writeInt(u16, @intCast(bools.len), .big);
            try writeBits(writer, bools);  // Bit-packed
        },
        .exprs => |e| {
            try writer.writeInt(u16, @intCast(e.items.len), .big);
            try serializeSType(e.elem_type, writer);
            for (e.items) |item| {
                try serializeExpr(item, writer);
            }
        },
    }
}
```

## Cost Model

```
Collection Operation Costs
─────────────────────────────────────────────────────

Operation       │ Cost Type    │ Formula
────────────────┼──────────────┼──────────────────────
length          │ Fixed        │ 10
apply(i)        │ Fixed        │ 10
get(i)          │ Fixed        │ 10
map(f)          │ PerItem      │ 10 + ⌈n/10⌉ × 5
filter(p)       │ PerItem      │ 20 + ⌈n/10⌉ × 5
fold(z, op)     │ PerItem      │ 10 + ⌈n/10⌉ × 5
exists(p)       │ PerItem      │ 10 + ⌈k/10⌉ × 5  (k=items checked)
forall(p)       │ PerItem      │ 10 + ⌈k/10⌉ × 5  (k=items checked)
slice(from,to)  │ PerItem      │ 10 + ⌈len/10⌉ × 2
append(other)   │ PerItem      │ 20 + ⌈(n+m)/10⌉ × 2
zip(other)      │ Fixed        │ 10 (structural)
unzip           │ Fixed        │ 10 (PairColl), PerItem (array)
flatMap(f)      │ Dynamic      │ depends on result sizes
─────────────────────────────────────────────────────

Where: n = collection size, k = items processed before short-circuit
```

## Set Operations

Distinct, union, intersection[^15]:

```zig
/// Set-like operations on collections
const CollSetOps = struct {

    /// Remove duplicates, preserving first occurrences
    pub fn distinct(coll: *const Coll, E: *Evaluator) !*Coll {
        var seen = std.AutoHashMap(Value, void).init(E.allocator);
        var result = std.ArrayList(Value).init(E.allocator);

        for (0..coll.length()) |i| {
            const elem = coll.get(i).?;
            if (!seen.contains(elem)) {
                try seen.put(elem, {});
                try result.append(elem);
            }
        }

        return coll.builder.fromArray(result.items, coll.elem_type);
    }

    /// Union preserving order (set semantics)
    pub fn unionSet(coll: *const Coll, other: *const Coll, E: *Evaluator) !*Coll {
        var seen = std.AutoHashMap(Value, void).init(E.allocator);
        var result = std.ArrayList(Value).init(E.allocator);

        // Add all from first collection
        for (0..coll.length()) |i| {
            const elem = coll.get(i).?;
            if (!seen.contains(elem)) {
                try seen.put(elem, {});
                try result.append(elem);
            }
        }

        // Add unseen from second collection
        for (0..other.length()) |i| {
            const elem = other.get(i).?;
            if (!seen.contains(elem)) {
                try seen.put(elem, {});
                try result.append(elem);
            }
        }

        return coll.builder.fromArray(result.items, coll.elem_type);
    }

    /// Multiset intersection
    pub fn intersect(coll: *const Coll, other: *const Coll, E: *Evaluator) !*Coll {
        // Count occurrences in other
        var counts = std.AutoHashMap(Value, usize).init(E.allocator);
        for (0..other.length()) |i| {
            const elem = other.get(i).?;
            const entry = try counts.getOrPut(elem);
            if (!entry.found_existing) {
                entry.value_ptr.* = 0;
            }
            entry.value_ptr.* += 1;
        }

        // Collect elements that exist in other
        var result = std.ArrayList(Value).init(E.allocator);
        for (0..coll.length()) |i| {
            const elem = coll.get(i).?;
            if (counts.get(elem)) |*count| {
                if (count.* > 0) {
                    try result.append(elem);
                    count.* -= 1;
                }
            }
        }

        return coll.builder.fromArray(result.items, coll.elem_type);
    }
};
```

## Summary

- **Coll[T]** is immutable, indexed, deterministic
- **CollOverArray** wraps arrays with specialized primitive support
- **PairColl** uses Structure-of-Arrays for O(1) unzip
- **CollBuilder** creates collections with automatic pair optimization
- **Short-circuit evaluation** for exists/forall reduces costs
- **Size limit** (100K elements) prevents DoS attacks
- All operations have defined costs for gas calculation

---

*Next: [Chapter 21: AVL Trees](./ch21-avl-trees.md)*

[^1]: Scala: [`Colls.scala:12-50`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/Colls.scala#L12-L50) (Coll trait)

[^2]: Rust: [`collection.rs:21-32`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/collection.rs#L21-L32) (Collection enum)

[^3]: Scala: [`Colls.scala:50-100`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/Colls.scala#L50-L100) (core operations)

[^4]: Rust: [`coll_by_index.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/coll_by_index.rs) (ByIndex)

[^5]: Scala: [`CollsOverArrays.scala:30-50`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/CollsOverArrays.scala#L30-L50) (map, filter)

[^6]: Rust: [`coll_map.rs:17-62`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/coll_map.rs#L17-L62) (Map struct)

[^7]: Rust: [`coll_exists.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/coll_exists.rs), [`coll_forall.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/coll_forall.rs)

[^8]: Scala: [`CollsOverArrays.scala:50-80`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/CollsOverArrays.scala#L50-L80) (slice, append)

[^9]: Scala: [`Colls.scala:150-180`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/Colls.scala#L150-L180) (PairColl trait)

[^10]: Scala: [`CollsOverArrays.scala:200-280`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/CollsOverArrays.scala#L200-L280) (PairOfCols)

[^11]: Scala: [`Colls.scala:180-220`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/Colls.scala#L180-L220) (CollBuilder trait)

[^12]: Scala: [`CollsOverArrays.scala:300-400`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/CollsOverArrays.scala#L300-L400) (CollOverArrayBuilder)

[^13]: Rust: [`collection.rs:34-56`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/collection.rs#L34-L56) (Collection::new)

[^14]: Rust: [`collection.rs:100-136`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/mir/collection.rs#L100-L136) (serialization)

[^15]: Scala: [`CollsOverArrays.scala:100-150`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/core/shared/src/main/scala/sigma/data/CollsOverArrays.scala#L100-L150) (set operations)
