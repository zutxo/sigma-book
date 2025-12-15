# Appendix B: Complete Opcode Table

This appendix provides the reference for all Opcodes used in ErgoTree serialization.

**Source**: `data/shared/src/main/scala/sigma/serialization/OpCodes.scala`

## Operations

| Decimal | Hex | Operation | Description |
|---------|-----|-----------|-------------|
| 113 | 0x71 | `TaggedVariable` | Reference to a context variable by ID |
| 114 | 0x72 | `ValUse` | Usage of a value defined by `ValDef` |
| 115 | 0x73 | `ConstantPlaceholder` | Reference to a segregated constant |
| 116 | 0x74 | `SubstConstants` | Substitute constants in a tree |
| 122 | 0x7A | `LongToByteArray` | Convert Long to Array[Byte] |
| 123 | 0x7B | `ByteArrayToBigInt` | Convert Array[Byte] to BigInt |
| 124 | 0x7C | `ByteArrayToLong` | Convert Array[Byte] to Long |
| 125 | 0x7D | `Downcast` | Downcast numeric type |
| 126 | 0x7E | `Upcast` | Upcast numeric type |
| 127 | 0x7F | `True` | Boolean true |
| 128 | 0x80 | `False` | Boolean false |
| 129 | 0x81 | `UnitConstant` | Unit value |
| 130 | 0x82 | `GroupGenerator` | Group generator point |
| 131 | 0x83 | `ConcreteCollection` | Collection of items |
| 133 | 0x85 | `ConcreteCollectionBooleanConstant` | Collection of booleans (optimized) |
| 134 | 0x86 | `Tuple` | Tuple creation |
| 135 | 0x87 | `Select1` | Tuple select 1st element |
| 136 | 0x88 | `Select2` | Tuple select 2nd element |
| 137 | 0x89 | `Select3` | Tuple select 3rd element |
| 138 | 0x8A | `Select4` | Tuple select 4th element |
| 139 | 0x8B | `Select5` | Tuple select 5th element |
| 140 | 0x8C | `SelectField` | Select field by index |
| 143 | 0x8F | `Lt` | Less than |
| 144 | 0x90 | `Le` | Less than or equal |
| 145 | 0x91 | `Gt` | Greater than |
| 146 | 0x92 | `Ge` | Greater than or equal |
| 147 | 0x93 | `Eq` | Equal |
| 148 | 0x94 | `Neq` | Not equal |
| 149 | 0x95 | `If` | If-Then-Else |
| 150 | 0x96 | `And` | Logical AND |
| 151 | 0x97 | `Or` | Logical OR |
| 152 | 0x98 | `AtLeast` | Threshold signature check |
| 153 | 0x99 | `Minus` | Subtraction |
| 154 | 0x9A | `Plus` | Addition |
| 155 | 0x9B | `Xor` | Logical XOR |
| 156 | 0x9C | `Multiply` | Multiplication |
| 157 | 0x9D | `Division` | Division |
| 158 | 0x9E | `Modulo` | Modulo |
| 159 | 0x9F | `Exponentiate` | Exponentiation (BigInt) |
| 160 | 0xA0 | `MultiplyGroup` | Group element multiplication |
| 161 | 0xA1 | `Min` | Minimum |
| 162 | 0xA2 | `Max` | Maximum |
| 163 | 0xA3 | `Height` | Current block height |
| 164 | 0xA4 | `Inputs` | Transaction inputs |
| 165 | 0xA5 | `Outputs` | Transaction outputs |
| 166 | 0xA6 | `LastBlockUtxoRootHash` | Last block UTXO root |
| 167 | 0xA7 | `Self` | Self box |
| 172 | 0xAC | `MinerPubkey` | Miner public key |
| 173 | 0xAD | `MapCollection` | Map over collection |
| 174 | 0xAE | `Exists` | Exists predicate |
| 175 | 0xAF | `ForAll` | ForAll predicate |
| 176 | 0xB0 | `Fold` | Fold collection |
| 177 | 0xB1 | `SizeOf` | Collection size |
| 178 | 0xB2 | `ByIndex` | Access by index |
| 179 | 0xB3 | `Append` | Append to collection |
| 180 | 0xB4 | `Slice` | Slice collection |
| 181 | 0xB5 | `Filter` | Filter collection |
| 182 | 0xB6 | `AvlTree` | AvlTree operations |
| 183 | 0xB7 | `AvlTreeGet` | AvlTree get |
| 184 | 0xB8 | `FlatMap` | FlatMap collection |
| 193 | 0xC1 | `ExtractAmount` | Box value |
| 194 | 0xC2 | `ExtractScriptBytes` | Box proposition bytes |
| 195 | 0xC3 | `ExtractBytes` | Box bytes |
| 196 | 0xC4 | `ExtractBytesWithNoRef` | Box bytes (no ref) |
| 197 | 0xC5 | `ExtractId` | Box ID |
| 198 | 0xC6 | `ExtractRegisterAs` | Get register value |
| 199 | 0xC7 | `ExtractCreationInfo` | Box creation info |
| 203 | 0xCB | `CalcBlake2b256` | Hash function |
| 204 | 0xCC | `CalcSha256` | Hash function |
| 205 | 0xCD | `ProveDlog` | Prove DLog |
| 206 | 0xCE | `ProveDiffieHellmanTuple` | Prove DHT |
| 207 | 0xCF | `SigmaPropIsProven` | Check if proven |
| 208 | 0xD0 | `SigmaPropBytes` | Serialize SigmaProp |
| 209 | 0xD1 | `BoolToSigmaProp` | Convert Bool to SigmaProp |
| 210 | 0xD2 | `TrivialPropFalse` | Always false |
| 211 | 0xD3 | `TrivialPropTrue` | Always true |
| 212 | 0xD4 | `DeserializeContext` | Deserialize from context |
| 213 | 0xD5 | `DeserializeRegister` | Deserialize from register |
| 214 | 0xD6 | `ValDef` | Define value |
| 215 | 0xD7 | `FunDef` | Define function |
| 216 | 0xD8 | `BlockValue` | Block expression |
| 217 | 0xD9 | `FuncValue` | Function literal |
| 218 | 0xDA | `FuncApply` | Apply function |
| 219 | 0xDB | `PropertyCall` | Call property |
| 220 | 0xDC | `MethodCall` | Call method |
| 221 | 0xDD | `Global` | Global object |
| 222 | 0xDE | `SomeValue` | Option Some |
| 223 | 0xDF | `NoneValue` | Option None |
| 227 | 0xE3 | `GetVar` | Get context variable |
| 228 | 0xE4 | `OptionGet` | Option.get |
| 229 | 0xE5 | `OptionGetOrElse` | Option.getOrElse |
| 230 | 0xE6 | `OptionIsDefined` | Option.isDefined |
| 231 | 0xE7 | `ModQ` | Modulo Q |
| 232 | 0xE8 | `PlusModQ` | Plus Mod Q |
| 233 | 0xE9 | `MinusModQ` | Minus Mod Q |
| 234 | 0xEA | `SigmaAnd` | Sigma AND |
| 235 | 0xEB | `SigmaOr` | Sigma OR |
| 236 | 0xEC | `BinOr` | Binary OR |
| 237 | 0xED | `BinAnd` | Binary AND |
| 238 | 0xEE | `DecodePoint` | Decode group element |
| 239 | 0xEF | `LogicalNot` | Logical NOT |
| 240 | 0xF0 | `Negation` | Numeric negation |
| 241 | 0xF1 | `BitInversion` | Bitwise inversion |
| 242 | 0xF2 | `BitOr` | Bitwise OR |
| 243 | 0xF3 | `BitAnd` | Bitwise AND |
| 244 | 0xF4 | `BinXor` | Binary XOR |
| 245 | 0xF5 | `BitXor` | Bitwise XOR |
| 246 | 0xF6 | `BitShiftRight` | Bitwise shift right |
| 247 | 0xF7 | `BitShiftLeft` | Bitwise shift left |
| 248 | 0xF8 | `BitShiftRightZeroed` | Logical shift right |
| 249 | 0xF9 | `CollShiftRight` | Collection shift right |
| 250 | 0xFA | `CollShiftLeft` | Collection shift left |
| 251 | 0xFB | `CollShiftRightZeroed` | Collection logic shift right |
| 252 | 0xFC | `CollRotateLeft` | Collection rotate left |
| 253 | 0xFD | `CollRotateRight` | Collection rotate right |
| 254 | 0xFE | `Context` | Context object |
| 255 | 0xFF | `XorOf` | XOR of collection |

---
*[Previous: Appendix A](./appendix-a-type-codes.md) | [Next: Appendix C](./appendix-c-costs.md)*
