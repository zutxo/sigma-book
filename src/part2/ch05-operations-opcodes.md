# Chapter 5: Operations and Opcodes

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Bytecode encoding concepts
- Prior chapters: [Chapter 4](./ch04-value-nodes.md), [Chapter 2](../part1/ch02-type-system.md)

## Learning Objectives

- Understand opcode encoding scheme
- Navigate the complete opcode space (0x00-0xFF)
- Identify operations by category
- Understand cost descriptors for operations

## Opcode Encoding Scheme

Every ErgoTree operation is identified by a single-byte opcode[^1][^2]:

```
Opcode Space Layout:
┌────────────────────────────────────────────────────────┐
│ 0x00       │ Reserved (Undefined)                      │
├────────────┼───────────────────────────────────────────┤
│ 0x01-0x70  │ Constant type codes (optimized encoding)  │
├────────────┼───────────────────────────────────────────┤
│ 0x71       │ Function type marker (LastConstantCode+1) │
├────────────┼───────────────────────────────────────────┤
│ 0x72-0xFF  │ Operation codes (newOpCode 1-143)         │
└────────────┴───────────────────────────────────────────┘
```

Constant values 0x01-0x70 are encoded by type code directly, saving one byte.

```zig
const OpCode = struct {
    value: u8,

    pub const FIRST_DATA_TYPE: u8 = 1;
    pub const LAST_DATA_TYPE: u8 = 111;
    pub const LAST_CONSTANT_CODE: u8 = 112; // LAST_DATA_TYPE + 1

    pub fn new(shift: u8) OpCode {
        return .{ .value = LAST_CONSTANT_CODE + shift };
    }

    pub fn isConstant(byte: u8) bool {
        return byte >= FIRST_DATA_TYPE and byte <= LAST_CONSTANT_CODE;
    }
};
```

## Opcode Definitions

```zig
const OpCodes = struct {
    // Variables (0x71-0x74)
    pub const TaggedVariable = OpCode.new(1);     // 113
    pub const ValUse = OpCode.new(2);             // 114
    pub const ConstantPlaceholder = OpCode.new(3); // 115
    pub const SubstConstants = OpCode.new(4);     // 116

    // Conversions (0x7A-0x7E)
    pub const LongToByteArray = OpCode.new(10);   // 122
    pub const ByteArrayToBigInt = OpCode.new(11); // 123
    pub const ByteArrayToLong = OpCode.new(12);   // 124
    pub const Downcast = OpCode.new(13);          // 125
    pub const Upcast = OpCode.new(14);            // 126

    // Literals (0x7F-0x86)
    pub const True = OpCode.new(15);              // 127
    pub const False = OpCode.new(16);             // 128
    pub const UnitConstant = OpCode.new(17);      // 129
    pub const GroupGenerator = OpCode.new(18);    // 130
    pub const Coll = OpCode.new(19);              // 131
    pub const CollOfBoolConst = OpCode.new(21);   // 133
    pub const Tuple = OpCode.new(22);             // 134

    // Tuple access (0x87-0x8C)
    pub const Select1 = OpCode.new(23);           // 135
    pub const Select2 = OpCode.new(24);           // 136
    pub const Select3 = OpCode.new(25);           // 137
    pub const Select4 = OpCode.new(26);           // 138
    pub const Select5 = OpCode.new(27);           // 139
    pub const SelectField = OpCode.new(28);       // 140

    // Relations (0x8F-0x98)
    pub const Lt = OpCode.new(31);                // 143
    pub const Le = OpCode.new(32);                // 144
    pub const Gt = OpCode.new(33);                // 145
    pub const Ge = OpCode.new(34);                // 146
    pub const Eq = OpCode.new(35);                // 147
    pub const Neq = OpCode.new(36);               // 148
    pub const If = OpCode.new(37);                // 149
    pub const And = OpCode.new(38);               // 150
    pub const Or = OpCode.new(39);                // 151
    pub const AtLeast = OpCode.new(40);           // 152

    // Arithmetic (0x99-0xA2)
    pub const Minus = OpCode.new(41);             // 153
    pub const Plus = OpCode.new(42);              // 154
    pub const Xor = OpCode.new(43);               // 155
    pub const Multiply = OpCode.new(44);          // 156
    pub const Division = OpCode.new(45);          // 157
    pub const Modulo = OpCode.new(46);            // 158
    pub const Exponentiate = OpCode.new(47);      // 159
    pub const MultiplyGroup = OpCode.new(48);     // 160
    pub const Min = OpCode.new(49);               // 161
    pub const Max = OpCode.new(50);               // 162

    // Context (0xA3-0xAC)
    pub const Height = OpCode.new(51);            // 163
    pub const Inputs = OpCode.new(52);            // 164
    pub const Outputs = OpCode.new(53);           // 165
    pub const LastBlockUtxoRootHash = OpCode.new(54); // 166
    pub const Self = OpCode.new(55);              // 167
    pub const MinerPubkey = OpCode.new(60);       // 172

    // Collections (0xAD-0xB8)
    pub const Map = OpCode.new(61);               // 173
    pub const Exists = OpCode.new(62);            // 174
    pub const ForAll = OpCode.new(63);            // 175
    pub const Fold = OpCode.new(64);              // 176
    pub const SizeOf = OpCode.new(65);            // 177
    pub const ByIndex = OpCode.new(66);           // 178
    pub const Append = OpCode.new(67);            // 179
    pub const Slice = OpCode.new(68);             // 180
    pub const Filter = OpCode.new(69);            // 181
    pub const AvlTree = OpCode.new(70);           // 182
    pub const FlatMap = OpCode.new(72);           // 184

    // Box access (0xC1-0xC7)
    pub const ExtractAmount = OpCode.new(81);     // 193
    pub const ExtractScriptBytes = OpCode.new(82); // 194
    pub const ExtractBytes = OpCode.new(83);      // 195
    pub const ExtractBytesWithNoRef = OpCode.new(84); // 196
    pub const ExtractId = OpCode.new(85);         // 197
    pub const ExtractRegisterAs = OpCode.new(86); // 198
    pub const ExtractCreationInfo = OpCode.new(87); // 199

    // Crypto (0xCB-0xD3)
    pub const CalcBlake2b256 = OpCode.new(91);    // 203
    pub const CalcSha256 = OpCode.new(92);        // 204
    pub const ProveDlog = OpCode.new(93);         // 205
    pub const ProveDHTuple = OpCode.new(94);      // 206
    pub const SigmaPropBytes = OpCode.new(96);    // 208
    pub const BoolToSigmaProp = OpCode.new(97);   // 209
    pub const TrivialFalse = OpCode.new(98);      // 210
    pub const TrivialTrue = OpCode.new(99);       // 211

    // Blocks (0xD4-0xDD)
    pub const DeserializeContext = OpCode.new(100); // 212
    pub const DeserializeRegister = OpCode.new(101); // 213
    pub const ValDef = OpCode.new(102);           // 214
    pub const FunDef = OpCode.new(103);           // 215
    pub const BlockValue = OpCode.new(104);       // 216
    pub const FuncValue = OpCode.new(105);        // 217
    pub const FuncApply = OpCode.new(106);        // 218
    pub const PropertyCall = OpCode.new(107);     // 219
    pub const MethodCall = OpCode.new(108);       // 220
    pub const Global = OpCode.new(109);           // 221

    // Options (0xDE-0xE6)
    pub const SomeValue = OpCode.new(110);        // 222
    pub const NoneValue = OpCode.new(111);        // 223
    pub const GetVar = OpCode.new(115);           // 227
    pub const OptionGet = OpCode.new(116);        // 228
    pub const OptionGetOrElse = OpCode.new(117);  // 229
    pub const OptionIsDefined = OpCode.new(118);  // 230

    // Sigma props (0xEA-0xED)
    pub const SigmaAnd = OpCode.new(122);         // 234
    pub const SigmaOr = OpCode.new(123);          // 235
    pub const BinOr = OpCode.new(124);            // 236
    pub const BinAnd = OpCode.new(125);           // 237

    // Bitwise (0xEE-0xFB)
    pub const DecodePoint = OpCode.new(126);      // 238
    pub const LogicalNot = OpCode.new(127);       // 239
    pub const Negation = OpCode.new(128);         // 240
    pub const BitInversion = OpCode.new(129);     // 241
    pub const BitOr = OpCode.new(130);            // 242
    pub const BitAnd = OpCode.new(131);           // 243
    pub const BinXor = OpCode.new(132);           // 244
    pub const BitXor = OpCode.new(133);           // 245
    pub const BitShiftRight = OpCode.new(134);    // 246
    pub const BitShiftLeft = OpCode.new(135);     // 247
    pub const BitShiftRightZeroed = OpCode.new(136); // 248

    // Special (0xFE-0xFF)
    pub const Context = OpCode.new(142);          // 254
    pub const XorOf = OpCode.new(143);            // 255
};
```

## Opcode Categories Summary

| Category | Range | Count | Description |
|----------|-------|-------|-------------|
| Variables | 113-116 | 4 | Variable references, placeholders |
| Conversions | 122-126 | 5 | Type conversions |
| Literals | 127-134 | 8 | Boolean, unit, collections |
| Tuple access | 135-140 | 6 | Field selection |
| Relations | 143-152 | 10 | Comparisons, conditionals |
| Arithmetic | 153-162 | 10 | Math operations |
| Context | 163-172 | 6 | Transaction context |
| Collections | 173-184 | 10 | Collection operations |
| Box access | 193-199 | 7 | Box property access |
| Crypto | 203-211 | 9 | Hashing, sigma props |
| Blocks | 212-221 | 10 | Definitions, lambdas |
| Options | 222-230 | 7 | Option operations |
| Sigma props | 234-237 | 4 | Sigma composition |
| Bitwise | 238-248 | 11 | Bit operations |

## Arithmetic Operations

Arithmetic operations use type-based costing[^3][^4]:

```zig
const ArithOp = struct {
    op_code: OpCode,
    left: *const Value,
    right: *const Value,

    pub fn eval(self: *const ArithOp, env: *const DataEnv, E: *Evaluator) !Any {
        const x = try self.left.eval(env, E);
        const y = try self.right.eval(env, E);

        const cost = switch (self.left.tpe) {
            .big_int, .unsigned_big_int => 30,
            else => 15,
        };
        E.addCost(FixedCost{ .value = cost }, self.op_code);

        return switch (self.op_code.value) {
            OpCodes.Plus.value => arithPlus(x, y, self.left.tpe),
            OpCodes.Minus.value => arithMinus(x, y, self.left.tpe),
            OpCodes.Multiply.value => arithMultiply(x, y, self.left.tpe),
            OpCodes.Division.value => arithDivision(x, y, self.left.tpe),
            OpCodes.Modulo.value => arithModulo(x, y, self.left.tpe),
            OpCodes.Min.value => arithMin(x, y, self.left.tpe),
            OpCodes.Max.value => arithMax(x, y, self.left.tpe),
            else => error.UnknownOpcode,
        };
    }
};

fn arithPlus(x: Any, y: Any, tpe: SType) Any {
    return switch (tpe) {
        .byte => .{ .byte = x.byte +% y.byte },
        .short => .{ .short = x.short +% y.short },
        .int => .{ .int = x.int +% y.int },
        .long => .{ .long = x.long +% y.long },
        .big_int => .{ .big_int = x.big_int.add(y.big_int) },
        else => unreachable,
    };
}
```

### Arithmetic Cost Table

| Operation | Primitive Cost | BigInt Cost |
|-----------|---------------|-------------|
| Plus (+) | 15 | 20 |
| Minus (-) | 15 | 20 |
| Multiply (*) | 15 | 30 |
| Division (/) | 15 | 30 |
| Modulo (%) | 15 | 30 |
| Min/Max | 15 | 20 |

## Relation Operations

Comparison operations[^5]:

```zig
const Relation = struct {
    op_code: OpCode,
    left: *const Value,
    right: *const Value,

    pub fn eval(self: *const Relation, env: *const DataEnv, E: *Evaluator) !bool {
        const lv = try self.left.eval(env, E);
        const rv = try self.right.eval(env, E);

        const cost: u32 = switch (self.op_code.value) {
            OpCodes.Eq.value, OpCodes.Neq.value => 3, // Equality cheap
            else => 15, // Ordering comparisons
        };
        E.addCost(FixedCost{ .value = cost }, self.op_code);

        return switch (self.op_code.value) {
            OpCodes.Lt.value => compare(lv, rv, self.left.tpe) < 0,
            OpCodes.Le.value => compare(lv, rv, self.left.tpe) <= 0,
            OpCodes.Gt.value => compare(lv, rv, self.left.tpe) > 0,
            OpCodes.Ge.value => compare(lv, rv, self.left.tpe) >= 0,
            OpCodes.Eq.value => equalValues(lv, rv),
            OpCodes.Neq.value => !equalValues(lv, rv),
            else => error.UnknownOpcode,
        };
    }
};
```

## Logical Operations

Short-circuit evaluation with per-item cost[^6]:

```zig
const LogicalAnd = struct {
    input: *const Value, // Collection[Boolean]

    pub const COST = PerItemCost{
        .base = 10,
        .per_chunk = 5,
        .chunk_size = 32,
    };

    pub fn eval(self: *const LogicalAnd, env: *const DataEnv, E: *Evaluator) !bool {
        const coll = try self.input.eval(env, E);
        const items = coll.coll.bools;

        var result = true;
        var i: usize = 0;

        // Short-circuit: stop on first false
        while (i < items.len and result) : (i += 1) {
            result = result and items[i];
        }

        // Cost based on actual items processed
        E.addSeqCost(COST, i, OpCodes.And);
        return result;
    }
};

const BinaryAnd = struct {
    left: *const Value,
    right: *const Value,

    pub const COST = FixedCost{ .value = 20 };

    pub fn eval(self: *const BinaryAnd, env: *const DataEnv, E: *Evaluator) !bool {
        const l = try self.left.eval(env, E);
        E.addCost(COST, OpCodes.BinAnd);

        // Short-circuit: don't evaluate right if left is false
        if (!l.boolean) return false;
        return (try self.right.eval(env, E)).boolean;
    }
};
```

## Cost Descriptors

Three cost descriptor types[^7]:

```zig
/// Fixed cost regardless of input
const FixedCost = struct {
    value: u32, // JitCost units
};

/// Cost scales with input size
const PerItemCost = struct {
    base: u32,       // Fixed overhead
    per_chunk: u32,  // Cost per chunk
    chunk_size: u32, // Items per chunk

    pub fn calculate(self: PerItemCost, n_items: usize) u32 {
        const chunks = (n_items + self.chunk_size - 1) / self.chunk_size;
        return self.base + @intCast(chunks) * self.per_chunk;
    }
};

/// Cost depends on operand type
const TypeBasedCost = struct {
    primitive_cost: u32,
    big_int_cost: u32,

    pub fn forType(self: TypeBasedCost, tpe: SType) u32 {
        return switch (tpe) {
            .big_int, .unsigned_big_int => self.big_int_cost,
            else => self.primitive_cost,
        };
    }
};
```

## Context Operations

Access transaction context[^8]:

```zig
const ContextOps = struct {
    pub const Height = struct {
        pub const COST = FixedCost{ .value = 26 };

        pub fn eval(_: *const @This(), _: *const DataEnv, E: *Evaluator) i32 {
            E.addCost(COST, OpCodes.Height);
            return E.context.pre_header.height;
        }
    };

    pub const Inputs = struct {
        pub const COST = FixedCost{ .value = 10 };

        pub fn eval(_: *const @This(), _: *const DataEnv, E: *Evaluator) []const Box {
            E.addCost(COST, OpCodes.Inputs);
            return E.context.inputs;
        }
    };

    pub const Outputs = struct {
        pub const COST = FixedCost{ .value = 10 };

        pub fn eval(_: *const @This(), _: *const DataEnv, E: *Evaluator) []const Box {
            E.addCost(COST, OpCodes.Outputs);
            return E.context.outputs;
        }
    };

    pub const SelfBox = struct {
        pub const COST = FixedCost{ .value = 10 };

        pub fn eval(_: *const @This(), _: *const DataEnv, E: *Evaluator) *const Box {
            E.addCost(COST, OpCodes.Self);
            return E.context.self_box;
        }
    };
};
```

## Box Property Access

Extract box properties[^9]:

```zig
const ExtractAmount = struct {
    box: *const Value,

    pub const COST = FixedCost{ .value = 12 };

    pub fn eval(self: *const ExtractAmount, env: *const DataEnv, E: *Evaluator) !i64 {
        const b = try self.box.eval(env, E);
        E.addCost(COST, OpCodes.ExtractAmount);
        return b.box.value;
    }
};

const ExtractId = struct {
    box: *const Value,

    pub const COST = FixedCost{ .value = 12 };

    pub fn eval(self: *const ExtractId, env: *const DataEnv, E: *Evaluator) ![32]u8 {
        const b = try self.box.eval(env, E);
        E.addCost(COST, OpCodes.ExtractId);
        return b.box.id();
    }
};

const ExtractRegisterAs = struct {
    box: *const Value,
    register_id: u4, // 0-9

    pub const COST = FixedCost{ .value = 12 };

    pub fn eval(self: *const ExtractRegisterAs, env: *const DataEnv, E: *Evaluator) !?Constant {
        const b = try self.box.eval(env, E);
        E.addCost(COST, OpCodes.ExtractRegisterAs);
        return b.box.registers[self.register_id];
    }
};
```

## Cryptographic Operations

Hash and sigma prop operations[^10]:

```zig
const CalcBlake2b256 = struct {
    input: *const Value, // Coll[Byte]

    pub const COST = PerItemCost{
        .base = 117,
        .per_chunk = 1,
        .chunk_size = 128,
    };

    pub fn eval(self: *const CalcBlake2b256, env: *const DataEnv, E: *Evaluator) ![32]u8 {
        const bytes = try self.input.eval(env, E);
        E.addSeqCost(COST, bytes.coll.bytes.len, OpCodes.CalcBlake2b256);

        var hasher = std.crypto.hash.blake2.Blake2b256.init(.{});
        hasher.update(bytes.coll.bytes);
        return hasher.finalResult();
    }
};

const CalcSha256 = struct {
    input: *const Value,

    pub const COST = PerItemCost{
        .base = 79,
        .per_chunk = 1,
        .chunk_size = 64,
    };

    pub fn eval(self: *const CalcSha256, env: *const DataEnv, E: *Evaluator) ![32]u8 {
        const bytes = try self.input.eval(env, E);
        E.addSeqCost(COST, bytes.coll.bytes.len, OpCodes.CalcSha256);

        var hasher = std.crypto.hash.sha2.Sha256.init(.{});
        hasher.update(bytes.coll.bytes);
        return hasher.finalResult();
    }
};
```

## Summary

- Opcodes are single bytes: 0x01-0x70 for constants, 0x71-0xFF for operations
- Operations organized into categories: variables, conversions, relations, arithmetic, context, collections, box access, crypto, blocks, options, sigma props, bitwise
- Cost descriptors: `FixedCost`, `PerItemCost`, `TypeBasedCost`
- Arithmetic operations have type-based costs (BigInt more expensive)
- Logical operations implement short-circuit evaluation with accurate cost tracking

---

*Next: [Chapter 6: Methods on Types](./ch06-methods-on-types.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/serialization/OpCodes.scala`

[^2]: Rust: `ergotree-ir/src/serialization/op_code.rs:10-100`

[^3]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala:704-827`

[^4]: Rust: `ergotree-ir/src/serialization/bin_op.rs`

[^5]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala:908-1100`

[^6]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala` (AND, OR)

[^7]: Scala: `core/shared/src/main/scala/sigma/ast/CostKind.scala`

[^8]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala` (context operations)

[^9]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala` (box accessors)

[^10]: Scala: `data/shared/src/main/scala/sigma/ast/trees.scala` (crypto operations)
