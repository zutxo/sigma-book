# Chapter 22: Box Model

## Prerequisites

- UTXO model basics
- ErgoTree structure ([Chapter 3](../part1/ch03-ergotree-structure.md))
- Collections ([Chapter 20](./ch20-collections.md))

## Learning Objectives

- Understand the Ergo box as the fundamental UTXO structure
- Learn the register-based data model (R0-R9)
- Master token representation and management
- Understand box ID computation and reference semantics
- Implement box serialization

## Box Architecture

Boxes are Ergo's state containers—the extended UTXO model[^1][^2]:

```
Box Structure
─────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────┐
│                     ErgoBox                         │
├─────────────────────────────────────────────────────┤
│  box_id: [32]u8          Blake2b256(serialize(box)) │
├─────────────────────────────────────────────────────┤
│                   Mandatory Registers               │
│  ┌───────────────────────────────────────────────┐  │
│  │ R0: Long           Monetary value (NanoErgs)  │  │
│  │ R1: ErgoTree       Guarding script            │  │
│  │ R2: Coll[Token]    Secondary tokens           │  │
│  │ R3: (Int, Bytes)   Creation info              │  │
│  └───────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────┤
│                Non-Mandatory Registers              │
│  ┌───────────────────────────────────────────────┐  │
│  │ R4-R9: Any         Application-defined data   │  │
│  └───────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────┤
│               Transaction Reference                 │
│  ┌───────────────────────────────────────────────┐  │
│  │ transaction_id: [32]u8    Creating tx hash    │  │
│  │ index: u16                Output index in tx  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

## Core Box Structure

```zig
const ErgoBox = struct {
    /// Blake2b256 hash of serialized box (computed)
    box_id: BoxId,
    /// Amount in NanoErgs (R0)
    value: BoxValue,
    /// Guarding script (R1)
    ergo_tree: ErgoTree,
    /// Secondary tokens (R2), up to MAX_TOKENS_COUNT
    tokens: ?BoundedVec(Token, 1, MAX_TOKENS_COUNT),
    /// Additional registers R4-R9
    additional_registers: NonMandatoryRegisters,
    /// Block height when transaction was created (part of R3)
    creation_height: u32,
    /// Transaction that created this box (part of R3)
    transaction_id: TxId,
    /// Output index in transaction (part of R3)
    index: u16,

    pub const MAX_TOKENS_COUNT: usize = 122;
    pub const MAX_BOX_SIZE: usize = 4096;
    pub const MAX_SCRIPT_SIZE: usize = 4096;

    /// Create new box, computing box_id from content
    pub fn init(
        value: BoxValue,
        ergo_tree: ErgoTree,
        tokens: ?BoundedVec(Token, 1, MAX_TOKENS_COUNT),
        additional_registers: NonMandatoryRegisters,
        creation_height: u32,
        transaction_id: TxId,
        index: u16,
    ) !ErgoBox {
        var box_with_zero_id = ErgoBox{
            .box_id = BoxId.zero(),
            .value = value,
            .ergo_tree = ergo_tree,
            .tokens = tokens,
            .additional_registers = additional_registers,
            .creation_height = creation_height,
            .transaction_id = transaction_id,
            .index = index,
        };
        box_with_zero_id.box_id = try box_with_zero_id.calcBoxId();
        return box_with_zero_id;
    }

    /// Compute box ID as Blake2b256 hash of serialized bytes
    fn calcBoxId(self: *const ErgoBox) !BoxId {
        const bytes = try self.sigmaSerialize();
        const hash = blake2b256(bytes);
        return BoxId{ .digest = hash };
    }

    /// Create box from candidate by adding transaction reference
    pub fn fromBoxCandidate(
        candidate: *const ErgoBoxCandidate,
        transaction_id: TxId,
        index: u16,
    ) !ErgoBox {
        return init(
            candidate.value,
            candidate.ergo_tree,
            candidate.tokens,
            candidate.additional_registers,
            candidate.creation_height,
            transaction_id,
            index,
        );
    }
};
```

## ErgoBoxCandidate

Before confirmation, boxes exist as candidates without transaction reference[^3][^4]:

```zig
/// Box before transaction confirmation (no tx reference yet)
const ErgoBoxCandidate = struct {
    /// Amount in NanoErgs
    value: BoxValue,
    /// Guarding script
    ergo_tree: ErgoTree,
    /// Secondary tokens
    tokens: ?BoundedVec(Token, 1, ErgoBox.MAX_TOKENS_COUNT),
    /// Additional registers R4-R9
    additional_registers: NonMandatoryRegisters,
    /// Declared creation height
    creation_height: u32,

    pub fn toBox(self: *const ErgoBoxCandidate, tx_id: TxId, index: u16) !ErgoBox {
        return ErgoBox.fromBoxCandidate(self, tx_id, index);
    }
};
```

## Register Model

Ten registers total—four mandatory, six application-defined[^5][^6]:

```
Register Layout
─────────────────────────────────────────────────────
 ID    Type                   Purpose
─────────────────────────────────────────────────────
 R0    Long                   Monetary value (NanoErgs)
 R1    Coll[Byte]             Serialized ErgoTree
 R2    Coll[(Coll[Byte],Long)] Secondary tokens
 R3    (Int, Coll[Byte])      (height, txId ++ index)
─────────────────────────────────────────────────────
 R4    Any                    Application data
 R5    Any                    Application data
 R6    Any                    Application data
 R7    Any                    Application data
 R8    Any                    Application data
 R9    Any                    Application data
─────────────────────────────────────────────────────

Note: R4-R9 must be densely packed.
      If R6 is used, R4 and R5 must also be present.
```

### Register ID Types

```zig
/// Register identifier (0-9)
const RegisterId = union(enum) {
    mandatory: MandatoryRegisterId,
    non_mandatory: NonMandatoryRegisterId,

    pub const R0 = RegisterId{ .mandatory = .r0 };
    pub const R1 = RegisterId{ .mandatory = .r1 };
    pub const R2 = RegisterId{ .mandatory = .r2 };
    pub const R3 = RegisterId{ .mandatory = .r3 };

    pub fn fromByte(value: u8) !RegisterId {
        if (value < 4) {
            return RegisterId{ .mandatory = @enumFromInt(value) };
        } else if (value <= 9) {
            return RegisterId{ .non_mandatory = @enumFromInt(value) };
        } else {
            return error.RegisterIdOutOfBounds;
        }
    }
};

/// Mandatory registers (R0-R3) - every box has these
const MandatoryRegisterId = enum(u8) {
    /// Monetary value in NanoErgs
    r0 = 0,
    /// Guarding script (serialized ErgoTree)
    r1 = 1,
    /// Secondary tokens
    r2 = 2,
    /// Transaction reference and creation height
    r3 = 3,
};

/// Non-mandatory registers (R4-R9) - application defined
const NonMandatoryRegisterId = enum(u8) {
    r4 = 4,
    r5 = 5,
    r6 = 6,
    r7 = 7,
    r8 = 8,
    r9 = 9,

    pub const START_INDEX: usize = 4;
    pub const END_INDEX: usize = 9;
    pub const NUM_REGS: usize = 6;
};
```

## Non-Mandatory Registers

Densely-packed storage for R4-R9[^7][^8]:

```zig
const NonMandatoryRegisters = struct {
    /// Registers stored as contiguous array (R4 at index 0)
    values: []RegisterValue,
    allocator: Allocator,

    pub const MAX_SIZE: usize = NonMandatoryRegisterId.NUM_REGS;

    pub fn empty() NonMandatoryRegisters {
        return .{ .values = &.{}, .allocator = undefined };
    }

    /// Create from map, ensuring dense packing
    pub fn fromMap(
        allocator: Allocator,
        map: std.AutoHashMap(NonMandatoryRegisterId, Constant),
    ) !NonMandatoryRegisters {
        const count = map.count();
        if (count > MAX_SIZE) return error.InvalidSize;

        // Verify dense packing: R4...R(4+count-1) must all be present
        var values = try allocator.alloc(RegisterValue, count);
        var i: usize = 0;
        while (i < count) : (i += 1) {
            const reg_id: NonMandatoryRegisterId = @enumFromInt(4 + i);
            const constant = map.get(reg_id) orelse
                return error.NonDenselyPacked;
            values[i] = RegisterValue{ .parsed = constant };
        }

        return .{ .values = values, .allocator = allocator };
    }

    /// Get register by ID, returns null if not present
    pub fn get(self: *const NonMandatoryRegisters, reg_id: NonMandatoryRegisterId) ?*const RegisterValue {
        const index = @intFromEnum(reg_id) - NonMandatoryRegisterId.START_INDEX;
        if (index >= self.values.len) return null;
        return &self.values[index];
    }

    /// Get as Constant, handling parse errors
    pub fn getConstant(self: *const NonMandatoryRegisters, reg_id: NonMandatoryRegisterId) !?Constant {
        const reg_val = self.get(reg_id) orelse return null;
        return try reg_val.asConstant();
    }
};

/// Register value—either parsed Constant or unparseable bytes
const RegisterValue = union(enum) {
    parsed: Constant,
    parsed_tuple: EvaluatedTuple,
    invalid: struct {
        bytes: []const u8,
        error_msg: []const u8,
    },

    pub fn asConstant(self: *const RegisterValue) !Constant {
        return switch (self.*) {
            .parsed => |c| c,
            .parsed_tuple => |t| t.toConstant(),
            .invalid => |inv| error.UnparseableRegister,
        };
    }
};
```

## Box ID Computation

Box ID is Blake2b256 hash of serialized content[^9][^10]:

```zig
const BoxId = struct {
    digest: [32]u8,

    pub const SIZE: usize = 32;

    pub fn zero() BoxId {
        return .{ .digest = [_]u8{0} ** 32 };
    }

    pub fn fromBytes(bytes: []const u8) !BoxId {
        if (bytes.len != SIZE) return error.InvalidLength;
        var result: BoxId = undefined;
        @memcpy(&result.digest, bytes);
        return result;
    }
};

/// Compute box ID from serialized box bytes
pub fn computeBoxId(box_bytes: []const u8) BoxId {
    return BoxId{ .digest = blake2b256(box_bytes) };
}
```

The ID includes transaction reference, making each box unique:

```
Box ID Computation
─────────────────────────────────────────────────────

┌──────────────────────────────────────────────────┐
│              Serialized Box Bytes                │
├──────────────────────────────────────────────────┤
│  value (VLQ)                                     │
│  ergo_tree (bytes)                               │
│  creation_height (VLQ)                           │
│  tokens_count (u8)                               │
│  tokens[] (token_id + amount)                    │
│  registers_count (u8)                            │
│  additional_registers[]                          │
│  transaction_id (32 bytes)                       │
│  index (2 bytes, big-endian)                     │
└──────────────────────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │   Blake2b256    │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  BoxId (32 B)   │
              └─────────────────┘
```

## Register Access

Get register value with type checking[^11][^12]:

```zig
/// Get any register value (R0-R9)
pub fn getRegister(box: *const ErgoBox, id: RegisterId) !?Constant {
    return switch (id) {
        .mandatory => |mid| switch (mid) {
            .r0 => Constant.fromLong(box.value.as_i64()),
            .r1 => Constant.fromBytes(try box.ergo_tree.serialize()),
            .r2 => Constant.fromTokens(box.tokensRaw()),
            .r3 => Constant.fromTuple(box.creationInfo()),
        },
        .non_mandatory => |nid| try box.additional_registers.getConstant(nid),
    };
}

/// Get tokens as raw (bytes, amount) pairs
pub fn tokensRaw(box: *const ErgoBox) []const struct { []const i8, i64 } {
    if (box.tokens) |tokens| {
        var result = allocator.alloc(@TypeOf(result[0]), tokens.len);
        for (tokens.items(), 0..) |token, i| {
            result[i] = .{ token.token_id.asVecI8(), token.amount.as_i64() };
        }
        return result;
    }
    return &.{};
}

/// Get creation info as (height, txId ++ index)
pub fn creationInfo(box: *const ErgoBox) struct { i32, []const i8 } {
    var bytes: [34]u8 = undefined; // 32-byte tx_id + 2-byte index
    @memcpy(bytes[0..32], &box.transaction_id.digest);
    std.mem.writeInt(u16, bytes[32..34], box.index, .big);
    return .{
        @intCast(box.creation_height),
        std.mem.bytesAsSlice(i8, &bytes),
    };
}
```

## ExtractRegisterAs (AST Node)

Register access in ErgoScript compiles to ExtractRegisterAs[^13][^14]:

```zig
/// Box.R0 - Box.R9 operations
const ExtractRegisterAs = struct {
    /// Input box expression
    input: *const Expr,
    /// Register index (0-9)
    register_id: i8,
    /// Expected element type (wrapped in Option)
    elem_tpe: SType,

    pub const OP_CODE = OpCode.new(0x6E); // EXTRACT_REGISTER_AS

    pub fn tpe(self: *const ExtractRegisterAs) SType {
        return SType.option(self.elem_tpe);
    }

    pub fn eval(self: *const ExtractRegisterAs, env: *Env, ctx: *Context) !Value {
        const ir_box = try self.input.eval(env, ctx);
        const box = ir_box.asBox() orelse return error.TypeMismatch;

        const id = RegisterId.fromByte(@intCast(self.register_id)) catch
            return error.RegisterIdOutOfBounds;

        const reg_val_opt = try box.getRegister(id);

        if (reg_val_opt) |constant| {
            // Type must match exactly
            if (!constant.tpe.equals(self.elem_tpe)) {
                return error.UnexpectedType;
            }
            return Value.some(constant.value);
        } else {
            return Value.none();
        }
    }
};
```

## Token Representation

Tokens are (id, amount) pairs stored in R2[^15][^16]:

```zig
const Token = struct {
    /// 32-byte token identifier
    token_id: TokenId,
    /// Token amount (positive i64)
    amount: TokenAmount,
};

const TokenId = struct {
    digest: [32]u8,

    pub const SIZE: usize = 32;
};

const TokenAmount = struct {
    value: u64,

    pub fn as_i64(self: TokenAmount) i64 {
        return @intCast(self.value);
    }
};

/// Bounded collection of tokens (1 to MAX_TOKENS)
const BoxTokens = BoundedVec(Token, 1, ErgoBox.MAX_TOKENS_COUNT);
```

Token minting rule:

```
Token Creation Rule
─────────────────────────────────────────────────────

A new token can be minted when:
  token_id == INPUTS(0).id

This ensures uniqueness: only the owner of a box
can create tokens with that box's ID as token ID.

┌─────────────┐     Spend      ┌─────────────────┐
│  Input Box  │ ─────────────► │   Output Box    │
│  id: ABC123 │                │  token: ABC123  │
└─────────────┘                │  amount: 1000   │
                               └─────────────────┘
```

## Box Serialization

```zig
/// Serialize box with optional token ID indexing
pub fn serializeBoxWithIndexedDigests(
    box_value: BoxValue,
    ergo_tree_bytes: []const u8,
    tokens: ?BoxTokens,
    additional_registers: *const NonMandatoryRegisters,
    creation_height: u32,
    token_ids_in_tx: ?*const IndexSet(TokenId),
    writer: anytype,
) !void {
    // Value (VLQ-encoded)
    try box_value.serialize(writer);

    // ErgoTree bytes
    try writer.writeAll(ergo_tree_bytes);

    // Creation height (VLQ-encoded)
    try writeVLQ(writer, creation_height);

    // Tokens
    const token_slice = if (tokens) |t| t.items() else &[_]Token{};
    try writer.writeByte(@intCast(token_slice.len));

    for (token_slice) |token| {
        if (token_ids_in_tx) |index_set| {
            // Write index into transaction's token list
            const idx = index_set.getIndex(token.token_id) orelse
                return error.TokenNotInIndex;
            try writeVLQ(writer, @intCast(idx));
        } else {
            // Write full 32-byte token ID
            try writer.writeAll(&token.token_id.digest);
        }
        try writeVLQ(writer, token.amount.value);
    }

    // Additional registers
    try additional_registers.serialize(writer);
}

/// Full ErgoBox serialization (adds tx reference)
pub fn serializeErgoBox(box: *const ErgoBox, writer: anytype) !void {
    const ergo_tree_bytes = try box.ergo_tree.serialize();

    try serializeBoxWithIndexedDigests(
        box.value,
        ergo_tree_bytes,
        box.tokens,
        &box.additional_registers,
        box.creation_height,
        null,
        writer,
    );

    // Transaction reference
    try writer.writeAll(&box.transaction_id.digest);
    try writer.writeInt(u16, box.index, .big);
}
```

## Size Limits

```
Box Constraints
─────────────────────────────────────────────────────
Limit                    Value     Notes
─────────────────────────────────────────────────────
Max box size             4 KB      Total serialized
Max tokens per box       122       Computed from size
Max registers            10        R0-R9
Max script size          4 KB      ErgoTree in R1
─────────────────────────────────────────────────────
```

```zig
const SigmaConstants = struct {
    pub const MAX_BOX_SIZE: usize = 4 * 1024;
    pub const MAX_TOKENS: usize = 122;
    pub const MAX_REGISTERS: usize = 10;
};
```

## Box Interface Methods

Methods available on Box type[^17][^18]:

```zig
const BoxMethods = struct {
    /// Box.value: Long - monetary value in NanoErgs
    pub fn value(box: *const ErgoBox) i64 {
        return box.value.as_i64();
    }

    /// Box.propositionBytes: Coll[Byte] - serialized script
    pub fn propositionBytes(box: *const ErgoBox) ![]const u8 {
        return try box.ergo_tree.serialize();
    }

    /// Box.bytes: Coll[Byte] - full serialized box
    pub fn bytes(box: *const ErgoBox) ![]const u8 {
        return try box.serialize();
    }

    /// Box.bytesWithoutRef: Coll[Byte] - without tx reference
    pub fn bytesWithoutRef(box: *const ErgoBox) ![]const u8 {
        const candidate = ErgoBoxCandidate{
            .value = box.value,
            .ergo_tree = box.ergo_tree,
            .tokens = box.tokens,
            .additional_registers = box.additional_registers,
            .creation_height = box.creation_height,
        };
        return try candidate.serialize();
    }

    /// Box.id: Coll[Byte] - 32-byte Blake2b256 hash
    pub fn id(box: *const ErgoBox) []const u8 {
        return &box.box_id.digest;
    }

    /// Box.creationInfo: (Int, Coll[Byte])
    pub fn creationInfo(box: *const ErgoBox) struct { i32, []const u8 } {
        return box.creationInfo();
    }

    /// Box.tokens: Coll[(Coll[Byte], Long)]
    pub fn tokens(box: *const ErgoBox) []const Token {
        return if (box.tokens) |t| t.items() else &.{};
    }

    /// Box.getReg[T](i: Int): Option[T]
    pub fn getReg(box: *const ErgoBox, comptime T: type, index: i32) !?T {
        const id = try RegisterId.fromByte(@intCast(index));
        const constant = try box.getRegister(id) orelse return null;
        return constant.extractAs(T);
    }
};
```

## Type-Safe Register Access

Three outcomes when accessing registers[^19][^20]:

```
Register Access Outcomes
─────────────────────────────────────────────────────

┌─────────────────────────────────────────────────┐
│             box.R4[Int]                         │
└─────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
┌──────────────┐ ┌──────────┐ ┌────────────────┐
│ R4 not set   │ │ R4 = Int │ │ R4 = Long      │
│              │ │          │ │ (wrong type)   │
└──────┬───────┘ └────┬─────┘ └───────┬────────┘
       │              │               │
       ▼              ▼               ▼
   None           Some(value)     ERROR!
                                  InvalidType
```

```zig
/// Type-safe register access with explicit error handling
pub fn extractRegisterAs(
    box: *const ErgoBox,
    register_id: i8,
    expected_type: SType,
) !?Value {
    const id = try RegisterId.fromByte(@intCast(register_id));
    const constant_opt = try box.getRegister(id);

    if (constant_opt) |constant| {
        if (!constant.tpe.equals(expected_type)) {
            return error.InvalidType;
        }
        return constant.value;
    }
    return null;
}
```

## Summary

- **Boxes** are immutable UTXO state containers with 10 registers
- **R0-R3** are mandatory (value, script, tokens, creation info)
- **R4-R9** are application-defined, must be densely packed
- **Box ID** is Blake2b256 hash of serialized content including tx reference
- **Tokens** stored in R2, max 122 per box, minted via first input's ID
- **Type-safe access** with three outcomes: None, Some(value), or InvalidType
- **4KB limit** on total box size

---

*Next: [Chapter 23: Interpreter Wrappers](../part8/ch23-interpreter-wrappers.md)*

[^1]: Scala: `data/shared/src/main/scala/org/ergoplatform/ErgoBox.scala:50-59`

[^2]: Rust: `ergotree-ir/src/chain/ergo_box.rs:38-80`

[^3]: Scala: `data/shared/src/main/scala/org/ergoplatform/ErgoBoxCandidate.scala:36-41`

[^4]: Rust: `ergotree-ir/src/chain/ergo_box.rs:225-248`

[^5]: Scala: `data/shared/src/main/scala/org/ergoplatform/ErgoBox.scala:154-168`

[^6]: Rust: `ergotree-ir/src/chain/ergo_box/register/id.rs:78-90`

[^7]: Scala: `data/shared/src/main/scala/org/ergoplatform/ErgoBox.scala` (additionalRegisters)

[^8]: Rust: `ergotree-ir/src/chain/ergo_box/register.rs:27-91`

[^9]: Scala: `data/shared/src/main/scala/org/ergoplatform/ErgoBox.scala:72-73`

[^10]: Rust: `ergotree-ir/src/chain/ergo_box.rs:149-153`

[^11]: Scala: `data/shared/src/main/scala/sigma/data/CBox.scala:77-94`

[^12]: Rust: `ergotree-ir/src/chain/ergo_box.rs:156-168`

[^13]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala:1263` (SBoxMethods)

[^14]: Rust: `ergotree-ir/src/mir/extract_reg_as.rs:18-57`

[^15]: Scala: `data/shared/src/main/scala/org/ergoplatform/ErgoBox.scala:119-130`

[^16]: Rust: `ergotree-ir/src/chain/ergo_box.rs:36-37` (BoxTokens)

[^17]: Scala: `core/shared/src/main/scala/sigma/SigmaDsl.scala:414-536`

[^18]: Rust: `ergotree-ir/src/chain/ergo_box.rs:120-198`

[^19]: Scala: `data/shared/src/main/scala/sigma/data/CBox.scala:20-74`

[^20]: Rust: `ergotree-interpreter/src/eval/extract_reg_as.rs:15-47`
