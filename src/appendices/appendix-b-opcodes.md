# Appendix B: Complete Opcode Table

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


Complete reference for all operation codes used in ErgoTree serialization[^1][^2].

## Opcode Ranges

```
Opcode Space Organization
══════════════════════════════════════════════════════════════════

Range           Usage                           Encoding
─────────────────────────────────────────────────────────────────
0x00            Reserved (invalid)              -
0x01-0x6F       Data types (constants)          Type code directly
0x70            LastConstantCode boundary       112
0x71-0xFF       Operations                      LastConstantCode + shift

Operation Categories:
─────────────────────────────────────────────────────────────────
0x71-0x79       Variables & references          ValUse, ConstPlaceholder
0x7A-0x7E       Type conversions               Upcast, Downcast
0x7F-0x8C       Constants & tuples             True, False, Tuple
0x8F-0x98       Relations & logic              Lt, Gt, Eq, And, Or
0x99-0xA2       Arithmetic                     Plus, Minus, Multiply
0xA3-0xAC       Context access                 HEIGHT, INPUTS, OUTPUTS
0xAD-0xB8       Collection operations          Map, Filter, Fold
0xC1-0xC7       Box extraction                 ExtractAmount, ExtractId
0xCB-0xD5       Crypto & serialization         Blake2b, ProveDlog
0xD6-0xE7       Blocks & functions             ValDef, FuncValue, Apply
0xEA-0xEB       Sigma operations               SigmaAnd, SigmaOr
0xEC-0xFF       Bitwise & misc                 BitOr, BitAnd, XorOf
```

## Zig Opcode Definition

```zig
const OpCode = enum(u8) {
    // Constants region: 0x01-0x70 (type codes)
    // Operations start at LAST_CONSTANT_CODE + 1 = 113

    // Variable references
    tagged_variable = 0x71,      // Context variable by ID
    val_use = 0x72,              // Reference to ValDef binding
    constant_placeholder = 0x73, // Segregated constant reference
    subst_constants = 0x74,      // Substitute constants in tree

    // Type conversions
    long_to_byte_array = 0x7A,
    byte_array_to_bigint = 0x7B,
    byte_array_to_long = 0x7C,
    downcast = 0x7D,
    upcast = 0x7E,

    // Primitive constants
    true_const = 0x7F,
    false_const = 0x80,
    unit_constant = 0x81,
    group_generator = 0x82,

    // Collection & tuple construction
    concrete_collection = 0x83,
    concrete_collection_bool = 0x85,
    tuple = 0x86,
    select_1 = 0x87,
    select_2 = 0x88,
    select_3 = 0x89,
    select_4 = 0x8A,
    select_5 = 0x8B,
    select_field = 0x8C,

    // Relational operations
    lt = 0x8F,
    le = 0x90,
    gt = 0x91,
    ge = 0x92,
    eq = 0x93,
    neq = 0x94,

    // Control flow & logic
    if_op = 0x95,
    and_op = 0x96,
    or_op = 0x97,
    atleast = 0x98,

    // Arithmetic
    minus = 0x99,
    plus = 0x9A,
    xor = 0x9B,
    multiply = 0x9C,
    division = 0x9D,
    modulo = 0x9E,
    exponentiate = 0x9F,
    multiply_group = 0xA0,
    min = 0xA1,
    max = 0xA2,

    // Context access
    height = 0xA3,
    inputs = 0xA4,
    outputs = 0xA5,
    last_block_utxo_root_hash = 0xA6,
    self_box = 0xA7,
    miner_pubkey = 0xAC,

    // Collection operations
    map_collection = 0xAD,
    exists = 0xAE,
    forall = 0xAF,
    fold = 0xB0,
    size_of = 0xB1,
    by_index = 0xB2,
    append = 0xB3,
    slice = 0xB4,
    filter = 0xB5,
    avl_tree = 0xB6,
    avl_tree_get = 0xB7,
    flat_map = 0xB8,

    // Box extraction
    extract_amount = 0xC1,
    extract_script_bytes = 0xC2,
    extract_bytes = 0xC3,
    extract_bytes_with_no_ref = 0xC4,
    extract_id = 0xC5,
    extract_register_as = 0xC6,
    extract_creation_info = 0xC7,

    // Cryptographic operations
    calc_blake2b256 = 0xCB,
    calc_sha256 = 0xCC,
    prove_dlog = 0xCD,
    prove_diffie_hellman_tuple = 0xCE,
    sigma_prop_is_proven = 0xCF,
    sigma_prop_bytes = 0xD0,
    bool_to_sigma_prop = 0xD1,
    trivial_prop_false = 0xD2,
    trivial_prop_true = 0xD3,

    // Deserialization
    deserialize_context = 0xD4,
    deserialize_register = 0xD5,

    // Block & function definitions
    val_def = 0xD6,
    fun_def = 0xD7,
    block_value = 0xD8,
    func_value = 0xD9,
    func_apply = 0xDA,
    property_call = 0xDB,
    method_call = 0xDC,
    global = 0xDD,

    // Option operations
    some_value = 0xDE,
    none_value = 0xDF,
    get_var = 0xE3,
    option_get = 0xE4,
    option_get_or_else = 0xE5,
    option_is_defined = 0xE6,

    // Modular arithmetic (deprecated in v5+)
    mod_q = 0xE7,
    plus_mod_q = 0xE8,
    minus_mod_q = 0xE9,

    // Sigma operations
    sigma_and = 0xEA,
    sigma_or = 0xEB,

    // Binary operations
    bin_or = 0xEC,
    bin_and = 0xED,
    decode_point = 0xEE,
    logical_not = 0xEF,
    negation = 0xF0,

    // Bitwise operations
    bit_inversion = 0xF1,
    bit_or = 0xF2,
    bit_and = 0xF3,
    bin_xor = 0xF4,
    bit_xor = 0xF5,
    bit_shift_right = 0xF6,
    bit_shift_left = 0xF7,
    bit_shift_right_zeroed = 0xF8,

    // Collection bitwise operations
    coll_shift_right = 0xF9,
    coll_shift_left = 0xFA,
    coll_shift_right_zeroed = 0xFB,
    coll_rotate_left = 0xFC,
    coll_rotate_right = 0xFD,

    // Misc
    context = 0xFE,
    xor_of = 0xFF,

    pub fn isConstant(code: u8) bool {
        return code >= 0x01 and code <= 0x70;
    }

    pub fn isOperation(code: u8) bool {
        return code > 0x70;
    }

    pub fn fromShift(shift: u8) OpCode {
        return @enumFromInt(0x70 + shift);
    }
};
```

## Variable & Reference Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0x71 | 113 | TaggedVariable | Reference context variable by ID |
| 0x72 | 114 | ValUse | Use value defined by ValDef |
| 0x73 | 115 | ConstantPlaceholder | Reference segregated constant |
| 0x74 | 116 | SubstConstants | Substitute constants in tree |

## Type Conversion Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0x7A | 122 | LongToByteArray | Long → Coll[Byte] (big-endian) |
| 0x7B | 123 | ByteArrayToBigInt | Coll[Byte] → BigInt |
| 0x7C | 124 | ByteArrayToLong | Coll[Byte] → Long |
| 0x7D | 125 | Downcast | Numeric downcast (may overflow) |
| 0x7E | 126 | Upcast | Numeric upcast (always safe) |

## Constants & Tuples

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0x7F | 127 | True | Boolean true constant |
| 0x80 | 128 | False | Boolean false constant |
| 0x81 | 129 | UnitConstant | Unit () value |
| 0x82 | 130 | GroupGenerator | EC generator point G |
| 0x83 | 131 | ConcreteCollection | Coll construction |
| 0x85 | 133 | ConcreteCollectionBool | Optimized Coll[Boolean] |
| 0x86 | 134 | Tuple | Tuple construction |
| 0x87-0x8B | 135-139 | Select1-5 | Tuple element access |
| 0x8C | 140 | SelectField | Select by field index |

## Relational & Logic Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0x8F | 143 | Lt | Less than (<) |
| 0x90 | 144 | Le | Less or equal (≤) |
| 0x91 | 145 | Gt | Greater than (>) |
| 0x92 | 146 | Ge | Greater or equal (≥) |
| 0x93 | 147 | Eq | Equal (==) |
| 0x94 | 148 | Neq | Not equal (≠) |
| 0x95 | 149 | If | If-then-else |
| 0x96 | 150 | And | Logical AND (&&) |
| 0x97 | 151 | Or | Logical OR (\|\|) |
| 0x98 | 152 | AtLeast | k-of-n threshold |

## Arithmetic Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0x99 | 153 | Minus | Subtraction |
| 0x9A | 154 | Plus | Addition |
| 0x9B | 155 | Xor | Byte-array XOR |
| 0x9C | 156 | Multiply | Multiplication |
| 0x9D | 157 | Division | Integer division |
| 0x9E | 158 | Modulo | Remainder |
| 0x9F | 159 | Exponentiate | BigInt exponentiation |
| 0xA0 | 160 | MultiplyGroup | EC point multiplication |
| 0xA1 | 161 | Min | Minimum |
| 0xA2 | 162 | Max | Maximum |

## Context Access Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xA3 | 163 | Height | Current block height |
| 0xA4 | 164 | Inputs | Transaction inputs (INPUTS) |
| 0xA5 | 165 | Outputs | Transaction outputs (OUTPUTS) |
| 0xA6 | 166 | LastBlockUtxoRootHash | UTXO tree root hash |
| 0xA7 | 167 | Self | Current box (SELF) |
| 0xAC | 172 | MinerPubkey | Miner's public key |
| 0xFE | 254 | Context | Context object |

## Collection Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xAD | 173 | MapCollection | Transform elements |
| 0xAE | 174 | Exists | Any element matches |
| 0xAF | 175 | ForAll | All elements match |
| 0xB0 | 176 | Fold | Reduce to single value |
| 0xB1 | 177 | SizeOf | Collection length |
| 0xB2 | 178 | ByIndex | Element at index |
| 0xB3 | 179 | Append | Concatenate collections |
| 0xB4 | 180 | Slice | Extract sub-collection |
| 0xB5 | 181 | Filter | Keep matching elements |
| 0xB6 | 182 | AvlTree | AVL tree construction |
| 0xB7 | 183 | AvlTreeGet | AVL tree lookup |
| 0xB8 | 184 | FlatMap | Map and flatten |

## Box Extraction Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xC1 | 193 | ExtractAmount | Box.value (nanoErgs) |
| 0xC2 | 194 | ExtractScriptBytes | Box.propositionBytes |
| 0xC3 | 195 | ExtractBytes | Box.bytes (full) |
| 0xC4 | 196 | ExtractBytesWithNoRef | Box.bytesWithoutRef |
| 0xC5 | 197 | ExtractId | Box.id (32 bytes) |
| 0xC6 | 198 | ExtractRegisterAs | Box.Rx[T] |
| 0xC7 | 199 | ExtractCreationInfo | Box.creationInfo |

## Cryptographic Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xCB | 203 | CalcBlake2b256 | Blake2b256 hash |
| 0xCC | 204 | CalcSha256 | SHA-256 hash |
| 0xCD | 205 | ProveDlog | DLog proposition |
| 0xCE | 206 | ProveDHTuple | DHT proposition |
| 0xCF | 207 | SigmaPropIsProven | Check proven |
| 0xD0 | 208 | SigmaPropBytes | Serialize SigmaProp |
| 0xD1 | 209 | BoolToSigmaProp | Bool → SigmaProp |
| 0xD2 | 210 | TrivialPropFalse | Always false |
| 0xD3 | 211 | TrivialPropTrue | Always true |
| 0xEE | 238 | DecodePoint | Bytes → GroupElement |

## Block & Function Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xD4 | 212 | DeserializeContext | Deserialize from context |
| 0xD5 | 213 | DeserializeRegister | Deserialize from register |
| 0xD6 | 214 | ValDef | Define value binding |
| 0xD7 | 215 | FunDef | Define function |
| 0xD8 | 216 | BlockValue | Block expression { } |
| 0xD9 | 217 | FuncValue | Lambda expression |
| 0xDA | 218 | FuncApply | Apply function |
| 0xDB | 219 | PropertyCall | Property access |
| 0xDC | 220 | MethodCall | Method invocation |
| 0xDD | 221 | Global | Global object |

## Option Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xDE | 222 | SomeValue | Some(x) construction |
| 0xDF | 223 | NoneValue | None construction |
| 0xE3 | 227 | GetVar | Get context variable |
| 0xE4 | 228 | OptionGet | Option.get (may fail) |
| 0xE5 | 229 | OptionGetOrElse | Option.getOrElse |
| 0xE6 | 230 | OptionIsDefined | Option.isDefined |

## Sigma Operations

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xEA | 234 | SigmaAnd | Sigma AND (∧) |
| 0xEB | 235 | SigmaOr | Sigma OR (∨) |

## Bitwise Operations (v6+)

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xEF | 239 | LogicalNot | Boolean NOT (!) |
| 0xF0 | 240 | Negation | Numeric negation (-x) |
| 0xF1 | 241 | BitInversion | Bitwise NOT (~) |
| 0xF2 | 242 | BitOr | Bitwise OR (\|) |
| 0xF3 | 243 | BitAnd | Bitwise AND (&) |
| 0xF4 | 244 | BinXor | Binary XOR |
| 0xF5 | 245 | BitXor | Bitwise XOR (^) |
| 0xF6 | 246 | BitShiftRight | Arithmetic right shift (>>) |
| 0xF7 | 247 | BitShiftLeft | Left shift (<<) |
| 0xF8 | 248 | BitShiftRightZeroed | Logical right shift (>>>) |

## Collection Bitwise Operations (v6+)

| Hex | Decimal | Operation | Description |
|-----|---------|-----------|-------------|
| 0xF9 | 249 | CollShiftRight | Collection shift right |
| 0xFA | 250 | CollShiftLeft | Collection shift left |
| 0xFB | 251 | CollShiftRightZeroed | Collection logical shift right |
| 0xFC | 252 | CollRotateLeft | Collection rotate left |
| 0xFD | 253 | CollRotateRight | Collection rotate right |
| 0xFF | 255 | XorOf | XOR of collection elements |

## Opcode Parsing

```zig
const OpCodeParser = struct {
    /// Parse opcode from byte, determining if constant or operation
    pub fn parse(byte: u8) ParseResult {
        if (byte == 0) return .invalid;
        if (byte <= 0x70) return .{ .constant = byte };
        return .{ .operation = @enumFromInt(byte) };
    }

    /// Check if opcode requires additional data
    pub fn hasPayload(op: OpCode) bool {
        return switch (op) {
            .val_use,
            .constant_placeholder,
            .tagged_variable,
            .extract_register_as,
            .by_index,
            .select_field,
            .method_call,
            .property_call,
            => true,
            else => false,
        };
    }

    const ParseResult = union(enum) {
        invalid,
        constant: u8,
        operation: OpCode,
    };
};
```

## Constants

```zig
const OpCodeConstants = struct {
    /// First valid data type code
    pub const FIRST_DATA_TYPE: u8 = 0x01;
    /// Last data type code
    pub const LAST_DATA_TYPE: u8 = 111; // 0x6F
    /// Boundary between constants and operations
    pub const LAST_CONSTANT_CODE: u8 = 112; // 0x70
    /// First operation code
    pub const FIRST_OP_CODE: u8 = 113; // 0x71
    /// Maximum opcode value
    pub const MAX_OP_CODE: u8 = 255; // 0xFF
};
```

---

*[Previous: Appendix A](./appendix-a-type-codes.md) | [Next: Appendix C](./appendix-c-costs.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/serialization/OpCodes.scala`

[^2]: Rust: `ergotree-ir/src/serialization/op_code.rs:14-203`