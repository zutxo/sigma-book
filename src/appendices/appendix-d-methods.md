# Appendix D: Method Reference

This appendix provides the reference for all methods available on each type.

**Source**: `data/shared/src/main/scala/sigma/ast/methods.scala`

## Numeric Types

### SByte, SShort, SInt, SLong

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `toByte` | `(): Byte` | 1 | Convert to Byte |
| `toShort` | `(): Short` | 2 | Convert to Short |
| `toInt` | `(): Int` | 3 | Convert to Int |
| `toLong` | `(): Long` | 4 | Convert to Long |
| `toBigInt` | `(): BigInt` | 5 | Convert to BigInt |

**Version 6+ Methods** (on `SByte`, `SShort`, `SInt`, `SLong`):

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `toBytes` | `(): Coll[Byte]` | 6 | Convert to byte array |
| `toBits` | `(): Coll[Boolean]` | 7 | Convert to bits |
| `bitwiseInverse` | `(): T` | 8 | Bitwise NOT |
| `bitwiseOr` | `(other: T): T` | 9 | Bitwise OR |
| `bitwiseAnd` | `(other: T): T` | 10 | Bitwise AND |
| `bitwiseXor` | `(other: T): T` | 11 | Bitwise XOR |
| `shiftLeft` | `(bits: Int): T` | 12 | Left shift |
| `shiftRight` | `(bits: Int): T` | 13 | Right shift (arithmetic) |

### SBigInt

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `toByte` | `(): Byte` | 1 | Convert to Byte |
| `toShort` | `(): Short` | 2 | Convert to Short |
| `toInt` | `(): Int` | 3 | Convert to Int |
| `toLong` | `(): Long` | 4 | Convert to Long |
| `toBigInt` | `(): BigInt` | 5 | Identity (returns self) |

**Version 6+ Methods**:

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `toBytes` | `(): Coll[Byte]` | 6 | 32-byte big-endian |
| `toBits` | `(): Coll[Boolean]` | 7 | 256 bits |
| `bitwiseInverse` | `(): BigInt` | 8 | Bitwise NOT |
| `bitwiseOr` | `(other: BigInt): BigInt` | 9 | Bitwise OR |
| `bitwiseAnd` | `(other: BigInt): BigInt` | 10 | Bitwise AND |
| `bitwiseXor` | `(other: BigInt): BigInt` | 11 | Bitwise XOR |
| `shiftLeft` | `(bits: Int): BigInt` | 12 | Left shift |
| `shiftRight` | `(bits: Int): BigInt` | 13 | Right shift |
| `toUnsigned` | `(): UnsignedBigInt` | 14 | Convert to unsigned |
| `toUnsignedMod` | `(m: UnsignedBigInt): UnsignedBigInt` | 15 | Modular unsigned |

### SUnsignedBigInt (v6+)

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `modInverse` | `(m: UnsignedBigInt): UnsignedBigInt` | 14 | Modular inverse |
| `plusMod` | `(other: UBI, m: UBI): UBI` | 15 | Modular addition |
| `subtractMod` | `(other: UBI, m: UBI): UBI` | 16 | Modular subtraction |
| `multiplyMod` | `(other: UBI, m: UBI): UBI` | 17 | Modular multiplication |
| `mod` | `(m: UnsignedBigInt): UnsignedBigInt` | 18 | Modulo |
| `toSigned` | `(): BigInt` | 19 | Convert to signed |

## SGroupElement

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `getEncoded` | `(): Coll[Byte]` | 2 | 33-byte compressed encoding |
| `exp` | `(k: BigInt): GroupElement` | 3 | Scalar multiplication |
| `multiply` | `(other: GroupElement): GroupElement` | 4 | Point addition |
| `negate` | `(): GroupElement` | 5 | Point negation |

**Version 6+ Methods**:

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `expUnsigned` | `(k: UnsignedBigInt): GroupElement` | 6 | Scalar multiply (unsigned) |

## SSigmaProp

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `propBytes` | `(): Coll[Byte]` | 1 | Serialized SigmaProp |
| `isProven` | `(): Boolean` | 2 | Check if proven (internal) |

## SBox

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `value` | `(): Long` | 1 | ERG amount in nanoERGs |
| `propositionBytes` | `(): Coll[Byte]` | 2 | Guarding script bytes |
| `bytes` | `(): Coll[Byte]` | 3 | Full serialized box |
| `bytesWithoutRef` | `(): Coll[Byte]` | 4 | Box bytes without creation ref |
| `id` | `(): Coll[Byte]` | 5 | 32-byte box ID |
| `creationInfo` | `(): (Int, Coll[Byte])` | 6 | (height, txId) |
| `getReg[T]` | `(regId: Int): Option[T]` | 7 | Get register value |
| `tokens` | `(): Coll[(Coll[Byte], Long)]` | 8 | Token pairs |
| `R0` - `R3` | `(): Option[T]` | 7 | Mandatory registers |
| `R4` - `R9` | `(): Option[T]` | 7 | Optional registers |

## SAvlTree

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `digest` | `(): Coll[Byte]` | 1 | 33-byte root digest |
| `enabledOperations` | `(): Byte` | 2 | Enabled operations flags |
| `keyLength` | `(): Int` | 3 | Key length in bytes |
| `valueLengthOpt` | `(): Option[Int]` | 4 | Optional value length |
| `isInsertAllowed` | `(): Boolean` | 5 | Insert flag |
| `isUpdateAllowed` | `(): Boolean` | 6 | Update flag |
| `isRemoveAllowed` | `(): Boolean` | 7 | Remove flag |
| `updateOperations` | `(newOps: Byte): AvlTree` | 8 | Update operations |
| `contains` | `(key: Coll[Byte], proof: Coll[Byte]): Boolean` | 9 | Check key exists |
| `get` | `(key: Coll[Byte], proof: Coll[Byte]): Option[Coll[Byte]]` | 10 | Get value |
| `getMany` | `(keys: Coll[Coll[Byte]], proof: Coll[Byte]): Coll[Option[Coll[Byte]]]` | 11 | Batch get |
| `insert` | `(entries: Coll[(Coll[Byte], Coll[Byte])], proof: Coll[Byte]): Option[AvlTree]` | 12 | Insert entries |
| `update` | `(operations: Coll[(Coll[Byte], Coll[Byte])], proof: Coll[Byte]): Option[AvlTree]` | 13 | Update entries |
| `remove` | `(keys: Coll[Coll[Byte]], proof: Coll[Byte]): Option[AvlTree]` | 14 | Remove entries |
| `updateDigest` | `(newDigest: Coll[Byte]): AvlTree` | 15 | Update digest |

## SContext

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `dataInputs` | `(): Coll[Box]` | 1 | Data input boxes |
| `headers` | `(): Coll[Header]` | 2 | Last 10 block headers |
| `preHeader` | `(): PreHeader` | 3 | Pre-header of current block |
| `INPUTS` | `(): Coll[Box]` | 4 | Transaction inputs |
| `OUTPUTS` | `(): Coll[Box]` | 5 | Transaction outputs |
| `HEIGHT` | `(): Int` | 6 | Current block height |
| `SELF` | `(): Box` | 7 | Self (current input box) |
| `selfBoxIndex` | `(): Int` | 8 | Index of SELF in inputs |
| `LastBlockUtxoRootHash` | `(): AvlTree` | 9 | UTXO state root |
| `minerPubKey` | `(): Coll[Byte]` | 10 | Miner's public key |
| `getVar[T]` | `(varId: Byte): Option[T]` | 11 | Get context variable |

## SHeader

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `id` | `(): Coll[Byte]` | 1 | Header ID |
| `version` | `(): Byte` | 2 | Block version |
| `parentId` | `(): Coll[Byte]` | 3 | Parent block ID |
| `ADProofsRoot` | `(): Coll[Byte]` | 4 | AD proofs root |
| `stateRoot` | `(): AvlTree` | 5 | State tree root |
| `transactionsRoot` | `(): Coll[Byte]` | 6 | Transactions root |
| `timestamp` | `(): Long` | 7 | Block timestamp |
| `nBits` | `(): Long` | 8 | Difficulty bits |
| `height` | `(): Int` | 9 | Block height |
| `extensionRoot` | `(): Coll[Byte]` | 10 | Extension root |
| `minerPk` | `(): GroupElement` | 11 | Miner public key |
| `powOnetimePk` | `(): GroupElement` | 12 | PoW one-time key |
| `powNonce` | `(): Coll[Byte]` | 13 | PoW nonce |
| `powDistance` | `(): BigInt` | 14 | PoW solution distance |
| `votes` | `(): Coll[Byte]` | 15 | Miner votes (3 bytes) |

**Version 6+ Methods**:

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `checkPow` | `(): Boolean` | 16 | Verify PoW solution |

## SPreHeader

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `version` | `(): Byte` | 1 | Block version |
| `parentId` | `(): Coll[Byte]` | 2 | Parent block ID |
| `timestamp` | `(): Long` | 3 | Block timestamp |
| `nBits` | `(): Long` | 4 | Difficulty bits |
| `height` | `(): Int` | 5 | Block height |
| `minerPk` | `(): GroupElement` | 6 | Miner public key |
| `votes` | `(): Coll[Byte]` | 7 | Miner votes |

## SGlobal

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `groupGenerator` | `(): GroupElement` | 1 | Generator point G |
| `xor` | `(left: Coll[Byte], right: Coll[Byte]): Coll[Byte]` | 2 | XOR byte arrays |

**Version 6+ Methods**:

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `serialize[T]` | `(value: T): Coll[Byte]` | 3 | Serialize value |
| `fromBigEndianBytes[T]` | `(bytes: Coll[Byte]): T` | 4 | Decode big-endian |
| `encodeNBits` | `(bi: BigInt): Long` | 5 | Encode as nBits |
| `decodeNBits` | `(nBits: Long): BigInt` | 6 | Decode nBits |
| `powHit` | `(k: Int, msg: Coll[Byte], nonce: Coll[Byte], h: Coll[Byte], N: BigInt): BigInt` | 7 | Autolykos2 PoW |

## SCollection

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `size` | `(): Int` | 1 | Collection size |
| `apply` | `(i: Int): T` | 2 | Element at index |
| `getOrElse` | `(i: Int, default: T): T` | 3 | Safe element access |
| `map[R]` | `(f: T => R): Coll[R]` | 4 | Transform elements |
| `exists` | `(p: T => Boolean): Boolean` | 5 | Any element matches |
| `fold[R]` | `(zero: R)(op: (R, T) => R): R` | 6 | Reduce collection |
| `forall` | `(p: T => Boolean): Boolean` | 7 | All elements match |
| `slice` | `(from: Int, until: Int): Coll[T]` | 8 | Extract range |
| `filter` | `(p: T => Boolean): Coll[T]` | 9 | Keep matching |
| `append` | `(other: Coll[T]): Coll[T]` | 10 | Concatenate |
| `indices` | `(): Coll[Int]` | 14 | Index collection |
| `flatMap[R]` | `(f: T => Coll[R]): Coll[R]` | 15 | Map and flatten |

**Version 6+ Methods**:

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `patch` | `(from: Int, patch: Coll[T], replaced: Int): Coll[T]` | 19 | Patch collection |
| `updated` | `(index: Int, elem: T): Coll[T]` | 20 | Update element |
| `updateMany` | `(indices: Coll[Int], values: Coll[T]): Coll[T]` | 21 | Batch update |
| `indexOf` | `(elem: T, from: Int): Int` | 26 | Find element |
| `zip[U]` | `(other: Coll[U]): Coll[(T, U)]` | 29 | Pair with other |
| `reverse` | `(): Coll[T]` | 36 | Reverse order |
| `startsWith` | `(prefix: Coll[T]): Boolean` | 37 | Check prefix |
| `endsWith` | `(suffix: Coll[T]): Boolean` | 38 | Check suffix |
| `get` | `(i: Int): Option[T]` | 39 | Safe element access |

## SOption

| Method | Signature | ID | Description |
|--------|-----------|-----|-------------|
| `isDefined` | `(): Boolean` | 2 | Check if Some |
| `get` | `(): T` | 3 | Unwrap (throws if None) |
| `getOrElse` | `(default: T): T` | 4 | Unwrap with default |
| `map[R]` | `(f: T => R): Option[R]` | 7 | Transform if Some |
| `filter` | `(p: T => Boolean): Option[T]` | 8 | Keep if predicate holds |

## STuple

Tuples support component access by position:

| Method | Signature | Description |
|--------|-----------|-------------|
| `_1` | First element |
| `_2` | Second element |
| `_N` | Nth element (up to 255) |

---
*[Previous: Appendix C](./appendix-c-costs.md) | [Next: Appendix E](./appendix-e-serialization.md)*
