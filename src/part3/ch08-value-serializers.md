# Chapter 8: Value Serializers

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- [Chapter 7](./ch07-serialization-framework.md) for VLQ encoding, type serialization, and `SigmaByteReader`/`SigmaByteWriter`
- [Chapter 4](../part2/ch04-value-nodes.md) for the `Value` hierarchy and expression node types
- [Chapter 5](../part2/ch05-operations-opcodes.md) for the opcode space and operation categories

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain opcode-based serialization dispatch and how it enables extensibility
- Implement value serializers following common patterns (binary, unary, nullary, collection)
- Describe constant extraction and placeholder substitution for segregated constant trees
- Handle type inference during deserialization using `ValDefTypeStore`

## Serialization Architecture

Chapter 7 covered the low-level encoding primitives (VLQ, ZigZag, type codes). This chapter builds on that foundation to show how entire expression trees are serialized. The key insight is that each expression's opcode determines its serialization format, enabling a registry-based dispatch pattern that scales to hundreds of operation types[^1][^2].

```
Expression Serialization Flow
─────────────────────────────────────────────────────────

        ┌─────────────────┐
        │   Expression    │
        └────────┬────────┘
                 │
    ┌────────────┴────────────┐
    │  Is Constant?           │
    └────────────┬────────────┘
           ┌─────┴─────┐
           │ Yes       │ No
           ▼           ▼
   ┌───────────────┐  ┌───────────────┐
   │ Extract to    │  │ Get OpCode    │
   │ Store or      │  │ Write OpCode  │
   │ Write Inline  │  │ Serialize Body│
   └───────────────┘  └───────────────┘
```

### Serializer Registry

All serializers are registered in a sparse array indexed by opcode[^3][^4]:

```zig
const ValueSerializer = struct {
    /// Sparse array of serializers indexed by opcode
    serializers: [256]?*const Serializer,

    pub fn init() ValueSerializer {
        var self = ValueSerializer{ .serializers = [_]?*const Serializer{null} ** 256 };

        // Constants
        self.register(OpCode.Constant, &ConstantSerializer);
        self.register(OpCode.ConstantPlaceholder, &ConstantPlaceholderSerializer);

        // Tuples
        self.register(OpCode.Tuple, &TupleSerializer);
        self.register(OpCode.SelectField, &SelectFieldSerializer);

        // Relations
        self.register(OpCode.GT, &BinOpSerializer);
        self.register(OpCode.GE, &BinOpSerializer);
        self.register(OpCode.LT, &BinOpSerializer);
        self.register(OpCode.LE, &BinOpSerializer);
        self.register(OpCode.EQ, &BinOpSerializer);
        self.register(OpCode.NEQ, &BinOpSerializer);

        // Logical
        self.register(OpCode.BinAnd, &BinOpSerializer);
        self.register(OpCode.BinOr, &BinOpSerializer);
        self.register(OpCode.BinXor, &BinOpSerializer);

        // Arithmetic
        self.register(OpCode.Plus, &BinOpSerializer);
        self.register(OpCode.Minus, &BinOpSerializer);
        self.register(OpCode.Multiply, &BinOpSerializer);
        self.register(OpCode.Division, &BinOpSerializer);
        self.register(OpCode.Modulo, &BinOpSerializer);

        // Context
        self.register(OpCode.Height, &NullarySerializer);
        self.register(OpCode.Self, &NullarySerializer);
        self.register(OpCode.Inputs, &NullarySerializer);
        self.register(OpCode.Outputs, &NullarySerializer);
        self.register(OpCode.Context, &NullarySerializer);
        self.register(OpCode.Global, &NullarySerializer);

        // Collections
        self.register(OpCode.Coll, &CollectionSerializer);
        self.register(OpCode.CollBoolConst, &BoolCollectionSerializer);
        self.register(OpCode.Map, &MapSerializer);
        self.register(OpCode.Filter, &FilterSerializer);
        self.register(OpCode.Fold, &FoldSerializer);

        // Method calls
        self.register(OpCode.PropertyCall, &PropertyCallSerializer);
        self.register(OpCode.MethodCall, &MethodCallSerializer);

        return self;
    }

    fn register(self: *ValueSerializer, opcode: OpCode, serializer: *const Serializer) void {
        self.serializers[opcode.value] = serializer;
    }

    pub fn getSerializer(self: *const ValueSerializer, opcode: OpCode) !*const Serializer {
        return self.serializers[opcode.value] orelse error.UnknownOpCode;
    }
};
```

## Serialization Dispatch

### Serialize Expression

```zig
pub fn serialize(expr: *const Expr, w: *SigmaByteWriter) !void {
    switch (expr.*) {
        .constant => |c| {
            if (w.constant_store) |store| {
                // Extract constant to store, write placeholder
                const idx = try store.put(c);
                try w.putByte(OpCode.ConstantPlaceholder.value);
                try w.putUInt(idx);
            } else {
                // Write constant inline (type + value)
                try ConstantSerializer.serialize(c, w);
            }
        },
        else => {
            const opcode = expr.opCode();
            try w.putByte(opcode.value);  // Write opcode first
            const ser = registry.getSerializer(opcode) catch return error.UnknownOpCode;
            try ser.serialize(expr, w);   // Then serialize body
        },
    }
}
```

### Deserialize Expression

```zig
pub fn deserialize(r: *SigmaByteReader) !Expr {
    const tag = try r.getByte();

    // Look-ahead: constants have type codes 1-112
    if (tag <= OpCode.LAST_CONSTANT_CODE) {
        return .{ .constant = try ConstantSerializer.deserializeWithTag(r, tag) };
    }

    const opcode = OpCode{ .value = tag };
    const ser = registry.getSerializer(opcode) catch {
        return error.UnknownOpCode;
    };
    return ser.deserialize(r);
}
```

## Constant Serialization

Constants are serialized as type followed by value[^5][^6]:

```zig
const ConstantSerializer = struct {
    pub fn serialize(c: Constant, w: *SigmaByteWriter) !void {
        try TypeSerializer.serialize(c.tpe, w);   // 1. Type
        try DataSerializer.serialize(c.value, c.tpe, w);  // 2. Value
    }

    pub fn deserialize(r: *SigmaByteReader) !Constant {
        const tag = try r.getByte();
        return deserializeWithTag(r, tag);
    }

    pub fn deserializeWithTag(r: *SigmaByteReader, tag: u8) !Constant {
        const tpe = try TypeSerializer.parseWithTag(r, tag);
        const value = try DataSerializer.deserialize(tpe, r);
        return Constant{ .tpe = tpe, .value = value };
    }
};
```

### Constant Placeholder

When constant segregation is enabled, constants become placeholders[^7]:

```zig
const ConstantPlaceholderSerializer = struct {
    pub fn serialize(ph: ConstantPlaceholder, w: *SigmaByteWriter) !void {
        try w.putUInt(ph.index);  // Just the index
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const id = try r.getUInt();

        if (r.substitute_placeholders) {
            // Return actual constant from store
            const c = try r.constant_store.get(@intCast(id));
            return .{ .constant = c };
        } else {
            // Return placeholder (for template extraction)
            const tpe = (try r.constant_store.get(@intCast(id))).tpe;
            return .{ .constant_placeholder = .{ .index = @intCast(id), .tpe = tpe } };
        }
    }
};
```

## Common Serializer Patterns

### BinOp Serializer (Two Arguments)

For binary operations like arithmetic and comparisons[^8]:

```zig
const BinOpSerializer = struct {
    pub fn serialize(expr: *const Expr, w: *SigmaByteWriter) !void {
        const binop = expr.asBinOp();
        try ValueSerializer.serialize(binop.left, w);   // Left operand
        try ValueSerializer.serialize(binop.right, w);  // Right operand
    }

    pub fn deserialize(r: *SigmaByteReader, kind: BinOp.Kind) !Expr {
        const left = try ValueSerializer.deserialize(r);
        const right = try ValueSerializer.deserialize(r);
        return .{ .bin_op = .{
            .kind = kind,
            .left = &left,
            .right = &right,
        } };
    }
};
```

### Unary Serializer (One Argument)

For single-input transformations:

```zig
const UnarySerializer = struct {
    pub fn serialize(input: *const Expr, w: *SigmaByteWriter) !void {
        try ValueSerializer.serialize(input, w);
    }

    pub fn deserialize(r: *SigmaByteReader) !*const Expr {
        return try ValueSerializer.deserialize(r);
    }
};
```

### Nullary Serializer (No Body)

For singletons where opcode is sufficient:

```zig
const NullarySerializer = struct {
    pub fn serialize(_: *const Expr, _: *SigmaByteWriter) !void {
        // Nothing to write - opcode is enough
    }

    pub fn deserialize(r: *SigmaByteReader, opcode: OpCode) !Expr {
        _ = r;
        return switch (opcode) {
            .Height => .{ .global_var = .height },
            .Self => .{ .global_var = .self_box },
            .Inputs => .{ .global_var = .inputs },
            .Outputs => .{ .global_var = .outputs },
            .Context => .context,
            .Global => .global,
            else => error.InvalidOpCode,
        };
    }
};
```

## Collection Serializers

### ConcreteCollection

For collections of expressions[^9]:

```zig
const CollectionSerializer = struct {
    const MAX_COLLECTION_ITEMS: u16 = 4096;  // DoS protection

    pub fn serialize(coll: *const Collection, w: *SigmaByteWriter) !void {
        try w.putUShort(@intCast(coll.items.len));  // Count
        try TypeSerializer.serialize(coll.elem_type, w);  // Element type
        for (coll.items) |item| {
            try ValueSerializer.serialize(item, w);  // Each item
        }
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const count = try r.getUShort();
        if (count > MAX_COLLECTION_ITEMS) return error.CollectionTooLarge;

        const elem_type = try TypeSerializer.deserialize(r);

        var items = try r.allocator.alloc(*Expr, count);
        for (0..count) |i| {
            items[i] = try ValueSerializer.deserialize(r);
        }

        return .{ .collection = .{
            .elem_type = elem_type,
            .items = items,
        } };
    }
};
// NOTE: In production, use a pre-allocated expression pool instead of
// dynamic allocation during deserialization. See ZIGMA_STYLE.md.
```

### Boolean Collection Constant

Compact serialization for Coll[Boolean] constants:

```zig
const BoolCollectionSerializer = struct {
    pub fn serialize(bools: []const bool, w: *SigmaByteWriter) !void {
        try w.putUShort(@intCast(bools.len));
        // Pack into bits
        const byte_count = (bools.len + 7) / 8;
        var i: usize = 0;
        for (0..byte_count) |_| {
            var byte: u8 = 0;
            for (0..8) |bit| {
                if (i < bools.len and bools[i]) {
                    byte |= @as(u8, 1) << @intCast(bit);
                }
                i += 1;
            }
            try w.putByte(byte);
        }
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const count = try r.getUShort();
        const byte_count = (count + 7) / 8;

        var bools = try r.allocator.alloc(bool, count);
        var i: usize = 0;
        for (0..byte_count) |_| {
            const byte = try r.getByte();
            for (0..8) |bit| {
                if (i >= count) break;
                bools[i] = (byte >> @intCast(bit)) & 1 == 1;
                i += 1;
            }
        }

        return .{ .coll_bool_const = bools };
    }
};
```

### Map/Filter/Fold

Higher-order collection operations:

```zig
const MapSerializer = struct {
    pub fn serialize(m: *const Map, w: *SigmaByteWriter) !void {
        try ValueSerializer.serialize(m.input, w);   // Collection
        try ValueSerializer.serialize(m.mapper, w);  // Function
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const input = try ValueSerializer.deserialize(r);
        const mapper = try ValueSerializer.deserialize(r);
        return .{ .map = .{ .input = &input, .mapper = &mapper } };
    }
};

const FoldSerializer = struct {
    pub fn serialize(f: *const Fold, w: *SigmaByteWriter) !void {
        try ValueSerializer.serialize(f.input, w);   // Collection
        try ValueSerializer.serialize(f.zero, w);    // Initial value
        try ValueSerializer.serialize(f.folder, w);  // Fold function
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const input = try ValueSerializer.deserialize(r);
        const zero = try ValueSerializer.deserialize(r);
        const folder = try ValueSerializer.deserialize(r);
        return .{ .fold = .{
            .input = &input,
            .zero = &zero,
            .folder = &folder,
        } };
    }
};
```

## Block and Function Serializers

### BlockValue

For blocks with local definitions[^10]:

```zig
const BlockValueSerializer = struct {
    pub fn serialize(block: *const BlockValue, w: *SigmaByteWriter) !void {
        try w.putUInt(block.items.len);  // Definition count
        for (block.items) |item| {
            try ValueSerializer.serialize(item, w);  // Each definition
        }
        try ValueSerializer.serialize(block.result, w);  // Result expression
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const count = try r.getUInt();
        var items = try r.allocator.alloc(*Expr, @intCast(count));
        for (0..count) |i| {
            items[i] = try ValueSerializer.deserialize(r);
        }
        const result = try ValueSerializer.deserialize(r);
        return .{ .block_value = .{ .items = items, .result = &result } };
    }
};
```

### FuncValue

For lambda functions:

```zig
const FuncValueSerializer = struct {
    pub fn serialize(func: *const FuncValue, w: *SigmaByteWriter) !void {
        try w.putUInt(func.args.len);  // Argument count
        for (func.args) |arg| {
            try w.putUInt(arg.id);      // Argument id
            try TypeSerializer.serialize(arg.tpe, w);  // Argument type
        }
        try ValueSerializer.serialize(func.body, w);  // Body
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const arg_count = try r.getUInt();
        var args = try r.allocator.alloc(FuncArg, @intCast(arg_count));

        for (0..arg_count) |i| {
            const id = try r.getUInt();
            const tpe = try TypeSerializer.deserialize(r);
            // Store type for ValUse resolution
            r.val_def_type_store.put(@intCast(id), tpe);
            args[i] = .{ .id = @intCast(id), .tpe = tpe };
        }

        const body = try ValueSerializer.deserialize(r);
        return .{ .func_value = .{ .args = args, .body = &body } };
    }
};
```

### ValDef / ValUse

Variable definitions and references:

```zig
const ValDefSerializer = struct {
    pub fn serialize(vd: *const ValDef, w: *SigmaByteWriter) !void {
        try w.putUInt(vd.id);
        try TypeSerializer.serialize(vd.tpe, w);
        try ValueSerializer.serialize(vd.rhs, w);
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const id = try r.getUInt();
        const tpe = try TypeSerializer.deserialize(r);
        // Store for ValUse resolution
        r.val_def_type_store.put(@intCast(id), tpe);
        const rhs = try ValueSerializer.deserialize(r);
        return .{ .val_def = .{ .id = @intCast(id), .tpe = tpe, .rhs = &rhs } };
    }
};

const ValUseSerializer = struct {
    pub fn serialize(vu: *const ValUse, w: *SigmaByteWriter) !void {
        try w.putUInt(vu.id);
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const id = try r.getUInt();
        // Lookup type from earlier ValDef
        const tpe = r.val_def_type_store.get(@intCast(id)) orelse
            return error.UndefinedVariable;
        return .{ .val_use = .{ .id = @intCast(id), .tpe = tpe } };
    }
};
```

## MethodCall Serializer

Method calls require type and method ID lookup[^11][^12]:

```zig
const MethodCallSerializer = struct {
    pub fn serialize(mc: *const MethodCall, w: *SigmaByteWriter) !void {
        try w.putByte(mc.method.obj_type.typeId());  // Type ID
        try w.putByte(mc.method.method_id);          // Method ID
        try ValueSerializer.serialize(mc.obj, w);    // Receiver
        try w.putUInt(mc.args.len);                  // Arg count
        for (mc.args) |arg| {
            try ValueSerializer.serialize(arg, w);   // Each argument
        }
        // Explicit type arguments (for generic methods)
        for (mc.method.explicit_type_args) |tvar| {
            const tpe = mc.type_subst.get(tvar) orelse continue;
            try TypeSerializer.serialize(tpe, w);
        }
    }

    pub fn deserialize(r: *SigmaByteReader) !Expr {
        const type_id = try r.getByte();
        const method_id = try r.getByte();
        const obj = try ValueSerializer.deserialize(r);

        const arg_count = try r.getUInt();
        var args = try r.allocator.alloc(*Expr, @intCast(arg_count));
        for (0..arg_count) |i| {
            args[i] = try ValueSerializer.deserialize(r);
        }

        // Lookup method by type and method ID
        const method = try SMethod.fromIds(type_id, method_id);

        // Check version compatibility
        if (r.tree_version.value < method.min_version.value) {
            return error.MethodNotAvailable;
        }

        // Read type arguments
        var type_args = std.AutoHashMap(STypeVar, SType).init(r.allocator);
        for (method.explicit_type_args) |tvar| {
            const tpe = try TypeSerializer.deserialize(r);
            try type_args.put(tvar, tpe);
        }

        return .{ .method_call = .{
            .obj = &obj,
            .method = method,
            .args = args,
            .type_subst = type_args,
        } };
    }
};
```

## Serializer Summary Table

```
OpCode Range    Category            Serializer Pattern
────────────────────────────────────────────────────────────
1-112           Constants           Type + Value inline
113             ConstPlaceholder    Index only
114-120         Global vars         Nullary (opcode only)
121-130         Unary ops           Single child
131-150         Binary ops          Left + Right
151-160         Collection ops      Input + Function
161-170         Block/Func          Items + Body
171-180         Method calls        TypeId + MethodId + Args
```

## Summary

This chapter covered the value serialization system that transforms ErgoTree expression trees to and from bytes:

- **Opcode dispatch** enables extensible serialization—the first byte of each expression determines which serializer handles the remaining bytes, allowing O(1) lookup via a sparse registry array
- **Constant extraction** supports two modes: inline serialization (type + value) when constant segregation is disabled, or placeholder indices when segregation is enabled for template sharing
- **Common serializer patterns** reduce code duplication: `BinOpSerializer` handles all two-argument operations, `UnarySerializer` handles single-input transformations, and `NullarySerializer` handles singletons where the opcode alone is sufficient
- **Collection serializers** include bounds checking to prevent DoS attacks from maliciously large collections during deserialization
- **Type inference** via `ValDefTypeStore` tracks variable types as `ValDef` nodes are deserialized, allowing `ValUse` nodes to recover their types without storing them redundantly
- **Method call serialization** includes type ID, method ID, and version checking to ensure compatibility with the ErgoTree version being deserialized

---

*Next: [Chapter 9: Elliptic Curve Cryptography](../part4/ch09-elliptic-curve-cryptography.md)*

[^1]: Scala: [`ValueSerializer.scala:65-95`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/ValueSerializer.scala#L65-L95)

[^2]: Rust: [`expr.rs:83-203`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/expr.rs#L83-L203)

[^3]: Scala: [`ValueSerializer.scala:50-182`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/ValueSerializer.scala#L50-L182)

[^4]: Rust: [`expr.rs:215-298`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/expr.rs#L215-L298)

[^5]: Scala: [`ConstantSerializer.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/ConstantSerializer.scala)

[^6]: Rust: [`constant.rs:9-29`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/constant.rs#L9-L29)

[^7]: Rust: [`constant_placeholder.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/constant_placeholder.rs)

[^8]: Rust: [`bin_op.rs`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/bin_op.rs)

[^9]: Scala: [`ConcreteCollectionSerializer.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/ConcreteCollectionSerializer.scala)

[^10]: Scala: [`BlockValueSerializer.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/BlockValueSerializer.scala)

[^11]: Scala: [`MethodCallSerializer.scala`](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/data/shared/src/main/scala/sigma/serialization/MethodCallSerializer.scala)

[^12]: Rust: [`method_call.rs:19-60`](https://github.com/ergoplatform/sigma-rust/blob/develop/ergotree-ir/src/serialization/method_call.rs#L19-L60)