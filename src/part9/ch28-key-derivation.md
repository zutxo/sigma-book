# Chapter 28: Key Derivation

## Prerequisites

- **Required knowledge**: Elliptic Curve Cryptography (Chapter 9), Hash Functions (Chapter 10)
- **Related concepts**: High-Level SDK (Chapter 27), Sigma Protocols (Chapter 11)
- **Prior chapters**: Chapter 27 (High-Level SDK)

## Learning Objectives

- Understand BIP-32 hierarchical deterministic key derivation
- Master derivation paths and their encoding
- Learn the difference between hardened and non-hardened derivation
- Understand EIP-3 key derivation for Ergo

## Source References

- `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/DerivationPath.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedKey.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedSecretKey.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/ExtendedPublicKey.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/Index.scala`

## Introduction

Hierarchical Deterministic (HD) wallets use a tree structure to derive an unlimited number of keys from a single master seed. This approach, defined in **BIP-32**, enables:

1. **Single backup** - One seed backs up all keys
2. **Organization** - Keys organized in logical hierarchy
3. **Privacy** - Fresh addresses for each transaction
4. **Watch-only wallets** - Public keys derived without secrets

## Key Derivation Overview

```
                     Master Seed
                         │
                    HMAC-SHA512
                         │
              ┌──────────┴──────────┐
              │                     │
         Master Key            Chain Code
              │                     │
              └─────────┬───────────┘
                        │
                  Master Extended Key
                        │
         ┌──────────────┼──────────────┐
         │              │              │
   m/44' (Purpose)  m/44'/429'   m/44'/429'/0'
         │          (Coin Type)  (Account)
         ▼              │              │
     Child Keys        ...           ...
```

## Index System

The `Index` object manages key indices and hardening:

```scala
// From Index.scala:5-16
object Index {
  val HardRangeStart = 0x80000000  // 2^31

  def hardIndex(i: Int): Int = i | HardRangeStart

  def isHardened(i: Int): Boolean = (i & HardRangeStart) != 0

  def serializeIndex(i: Int): Array[Byte] = ByteVector.fromInt(i).toArray

  def parseIndex(xs: Array[Byte]): Int = ByteVector(xs).toInt(signed = false)
}
```

### Hardened vs Non-Hardened

| Type | Index Range | Notation | Security |
|------|-------------|----------|----------|
| Non-hardened | 0 to 2³¹-1 | `0`, `1`, `2` | Public derivation possible |
| Hardened | 2³¹ to 2³²-1 | `0'`, `1'`, `2'` | Requires private key |

**Key insight**: Hardened derivation uses the private key in HMAC input, preventing public key derivation from parent public key.

## Derivation Path

The `DerivationPath` class represents a path in the key tree:

```scala
// From DerivationPath.scala:10-29
final case class DerivationPath(decodedPath: Seq[Int], publicBranch: Boolean) {

  def depth: Int = decodedPath.length

  /** Last element of the path, e.g. 2 for m/1/2 */
  def index: Int = decodedPath.last

  def isMaster: Boolean = depth == 1

  /** Encode as parsable string like "m/44'/429'/0'/0/0" */
  def encoded: String = {
    val masterPrefix = if (publicBranch) "M/" else "m/"
    val tailPath = decodedPath.tail
      .map(x => if (Index.isHardened(x)) s"${x - Index.HardRangeStart}'" else x.toString)
      .mkString("/")
    masterPrefix + tailPath
  }

  def extended(idx: Int): DerivationPath =
    DerivationPath(decodedPath :+ idx, publicBranch)

  def increased: DerivationPath =
    DerivationPath(decodedPath.dropRight(1) :+ (index + 1), publicBranch)
}
```

### Path Notation

- `m` - Private master key (lowercase)
- `M` - Public master key (uppercase)
- `'` - Hardened derivation
- `/` - Level separator

Examples:
- `m/44'/429'/0'/0/0` - First EIP-3 address
- `m/1` - Pre-EIP-3 first key
- `M/44'/429'/0'/0` - Public branch for watch-only

### Parsing Paths

```scala
// From DerivationPath.scala:70-84
def fromEncoded(path: String): Try[DerivationPath] = {
  val split = path.split("/")
  if (!split.headOption.exists(Seq("M", "m").contains)) {
    Failure(new Exception("Wrong path format"))
  } else {
    val pathTry = split.tail.foldLeft(Try(List(0))) { case (accTry, sym) =>
      accTry.flatMap { acc =>
        Try(
          if (sym.endsWith("'")) Index.hardIndex(sym.dropRight(1).toInt)
          else sym.toInt
        ).map(acc :+ _)
      }
    }
    val isPublicBranch = split.head == "M"
    pathTry.map(DerivationPath(_, isPublicBranch))
  }
}
```

## EIP-3 Derivation

Ergo's EIP-3 defines standard derivation paths:

```scala
// From Constants.scala:31-36
val preEip3DerivationPath: DerivationPath = DerivationPath.fromEncoded("m/1").get

val eip3DerivationPath: DerivationPath = DerivationPath.fromEncoded("m/44'/429'/0'/0/0").get
```

### EIP-3 Path Structure

```
m / 44' / 429' / 0' / 0 / 0
│    │      │     │   │   │
│    │      │     │   │   └── Address Index (non-hardened)
│    │      │     │   └────── Change (0=external, 1=internal)
│    │      │     └────────── Account Index (hardened)
│    │      └──────────────── Coin Type: 429 for Ergo
│    └─────────────────────── Purpose: BIP-44
└──────────────────────────── Master private key
```

### EIP-3 Detection

```scala
// From DerivationPath.scala:54-56
def isEip3: Boolean = {
  decodedPath.tail.startsWith(Constants.eip3DerivationPath.decodedPath.tail.take(3))
}
```

A path is EIP-3 compliant if it starts with `44'/429'/0'`.

## Extended Key Trait

The base trait for HD keys:

```scala
// From ExtendedKey.scala:16-43
trait ExtendedKey[T <: ExtendedKey[T]] {

  val path: DerivationPath

  def selfReflection: T

  /** Derive child key at index */
  def child(idx: Int): T

  /** Derive key at specified path */
  def derive(upPath: DerivationPath): T = {
    require(
      upPath.depth >= path.depth &&
        upPath.decodedPath.take(path.depth).zip(path.decodedPath)
          .forall { case (i1, i2) => i1 == i2 } &&
        upPath.publicBranch == path.publicBranch,
      s"Incompatible paths: $upPath, $path"
    )
    upPath.decodedPath.drop(path.depth)
      .foldLeft(selfReflection)((parent, i) => parent.child(i))
  }
}
```

Key properties from BIP-32:
- Extended keys include both key material and chain code (32 bytes each)
- Each extended key can derive 2³¹ normal children and 2³¹ hardened children
- Normal children use indices 0 to 2³¹-1
- Hardened children use indices 2³¹ to 2³²-1

## Extended Secret Key

The secret key implementation with derivation:

```scala
// From ExtendedSecretKey.scala:13-49
final class ExtendedSecretKey(
    val keyBytes: Array[Byte],      // 32-byte secret key
    val chainCode: Array[Byte],     // 32-byte chain code
    val usePre1627KeyDerivation: Boolean,
    val path: DerivationPath
) extends ExtendedKey[ExtendedSecretKey] with SecretKey {

  def selfReflection: ExtendedSecretKey = this

  override def privateInput: DLogProverInput =
    DLogProverInput(BigIntegers.fromUnsignedByteArray(keyBytes))

  def publicImage: ProveDlog = privateInput.publicImage

  def child(idx: Int): ExtendedSecretKey =
    ExtendedSecretKey.deriveChildSecretKey(this, idx)

  def publicKey: ExtendedPublicKey =
    new ExtendedPublicKey(
      CryptoFacade.getASN1Encoding(privateInput.publicImage.value, true),
      chainCode,
      path.toPublicBranch
    )

  def isErased: Boolean = keyBytes.forall(_ == 0x00)

  def zeroSecret(): Unit = java.util.Arrays.fill(keyBytes, 0: Byte)
}
```

### Child Secret Key Derivation

The core derivation algorithm from BIP-32:

```scala
// From ExtendedSecretKey.scala:53-78
@tailrec
def deriveChildSecretKey(parentKey: ExtendedSecretKey, idx: Int): ExtendedSecretKey = {
  // Data for HMAC:
  // - Hardened: 0x00 || parent_key (33 bytes)
  // - Normal: parent_public_key (33 bytes)
  val keyCoded: Array[Byte] =
    if (Index.isHardened(idx)) (0x00: Byte) +: parentKey.keyBytes
    else CryptoFacade.getASN1Encoding(parentKey.privateInput.publicImage.value, true)

  // HMAC-SHA512(chainCode, keyCoded || index)
  val (childKeyProto, childChainCode) = CryptoFacade
      .hashHmacSHA512(parentKey.chainCode, keyCoded ++ Index.serializeIndex(idx))
      .splitAt(CryptoFacade.SecretKeyLength)  // Split at 32 bytes

  val childKeyProtoDecoded = BigIntegers.fromUnsignedByteArray(childKeyProto)

  // child_key = (childKeyProto + parent_key) mod n
  val childKey = childKeyProtoDecoded
    .add(BigIntegers.fromUnsignedByteArray(parentKey.keyBytes))
    .mod(CryptoConstants.groupOrder)

  // Invalid keys: skip to next index (recursive)
  if (childKeyProtoDecoded.compareTo(CryptoConstants.groupOrder) >= 0 ||
      childKey.equals(BigInteger.ZERO))
    deriveChildSecretKey(parentKey, idx + 1)
  else {
    val keyBytes = if (parentKey.usePre1627KeyDerivation) {
      // Bug: may be less than 32 bytes (issue #1627)
      BigIntegers.asUnsignedByteArray(childKey)
    } else {
      // Correct: padded to 32 bytes
      BigIntegers.asUnsignedByteArray(CryptoFacade.SecretKeyLength, childKey)
    }
    new ExtendedSecretKey(keyBytes, childChainCode,
      parentKey.usePre1627KeyDerivation, parentKey.path.extended(idx))
  }
}
```

### Master Key Derivation

From seed to master extended key:

```scala
// From ExtendedSecretKey.scala:93-98
def deriveMasterKey(seed: Array[Byte], usePre1627KeyDerivation: Boolean): ExtendedSecretKey = {
  // HMAC-SHA512("Bitcoin seed", seed) → (master_key, chain_code)
  val (masterKey, chainCode) = CryptoFacade
      .hashHmacSHA512(CryptoFacade.BitcoinSeed, seed)
      .splitAt(CryptoFacade.SecretKeyLength)
  new ExtendedSecretKey(masterKey, chainCode, usePre1627KeyDerivation, DerivationPath.MasterPath)
}
```

The "Bitcoin seed" constant ensures compatibility with existing HD wallet implementations.

## Extended Public Key

Public key derivation (non-hardened only):

```scala
// From ExtendedPublicKey.scala:13-41
final class ExtendedPublicKey(
    val keyBytes: Array[Byte],   // 33-byte compressed public key
    val chainCode: Array[Byte], // 32-byte chain code
    val path: DerivationPath
) extends ExtendedKey[ExtendedPublicKey] {

  def selfReflection: ExtendedPublicKey = this

  def key: ProveDlog = ProveDlog(
    CryptoConstants.dlogGroup.ctx.decodePoint(keyBytes)
  )

  def child(idx: Int): ExtendedPublicKey =
    ExtendedPublicKey.deriveChildPublicKey(this, idx)
}
```

### Child Public Key Derivation

```scala
// From ExtendedPublicKey.scala:46-59
@tailrec
def deriveChildPublicKey(parentKey: ExtendedPublicKey, idx: Int): ExtendedPublicKey = {
  require(!Index.isHardened(idx), "Hardened public keys derivation is not supported")

  // HMAC-SHA512(chainCode, public_key || index)
  val (childKeyProto, childChainCode) = CryptoFacade
      .hashHmacSHA512(parentKey.chainCode, parentKey.keyBytes ++ Index.serializeIndex(idx))
      .splitAt(CryptoFacade.SecretKeyLength)

  val childKeyProtoDecoded = BigIntegers.fromUnsignedByteArray(childKeyProto)

  // child_public = point(childKeyProto) + parent_public
  val childKey = CryptoFacade.multiplyPoints(
    DLogProverInput(childKeyProtoDecoded).publicImage.value,
    parentKey.key.value
  )

  // Invalid keys: skip to next index
  if (childKeyProtoDecoded.compareTo(CryptoConstants.groupOrder) >= 0 ||
      CryptoFacade.isInfinityPoint(childKey)) {
    deriveChildPublicKey(parentKey, idx + 1)
  } else {
    new ExtendedPublicKey(
      CryptoFacade.getASN1Encoding(childKey, true),
      childChainCode,
      parentKey.path.extended(idx)
    )
  }
}
```

**Why hardened derivation requires private key**:
- Non-hardened: HMAC input = parent_public_key || index
- Hardened: HMAC input = 0x00 || parent_private_key || index

Without the private key, hardened derivation cannot be computed.

## Mnemonic to Master Key

Converting BIP-39 mnemonic to master key:

```scala
// From JavaHelpers.scala:282-301
def mnemonicToSeed(mnemonic: String, passOpt: Option[String] = None): Array[Byte] = {
  val normalizedMnemonic = CryptoFacade.normalizeChars(mnemonic.toCharArray)
  val normalizedPass = CryptoFacade.normalizeChars(
    s"mnemonic${passOpt.getOrElse("")}".toCharArray
  )
  CryptoFacade.generatePbkdf2Key(normalizedMnemonic, normalizedPass)
}

def seedToMasterKey(
    seedPhrase: SecretString,
    pass: SecretString = null,
    usePre1627KeyDerivation: Boolean
): ExtendedSecretKey = {
  val passOpt = if (pass == null || pass.isEmpty()) None else Some(pass.toStringUnsecure)
  val seed = mnemonicToSeed(seedPhrase.toStringUnsecure, passOpt)
  val masterKey = ExtendedSecretKey.deriveMasterKey(seed, usePre1627KeyDerivation)
  masterKey
}
```

The mnemonic-to-seed conversion uses PBKDF2-HMAC-SHA512 with 2048 iterations.

## Secret Key Types

The SDK supports multiple secret types:

```scala
// From SecretKey.scala:9-39
trait SecretKey {
  def privateInput: SigmaProtocolPrivateInput[_]
}

sealed trait PrimitiveSecretKey extends SecretKey

object PrimitiveSecretKey {
  def apply(sigmaPrivateInput: SigmaProtocolPrivateInput[_]): PrimitiveSecretKey =
    sigmaPrivateInput match {
      case dls: DLogProverInput => DlogSecretKey(dls)
      case dhts: DiffieHellmanTupleProverInput => DhtSecretKey(dhts)
    }
}

/** Discrete log secret: h = g^w */
case class DlogSecretKey(override val privateInput: DLogProverInput)
  extends PrimitiveSecretKey

/** Diffie-Hellman tuple secret: u = g^w and v = h^w */
case class DhtSecretKey(override val privateInput: DiffieHellmanTupleProverInput)
  extends PrimitiveSecretKey
```

## Path Navigation

### Finding Next Path

```scala
// From DerivationPath.scala:91-129
def nextPath(
    secrets: IndexedSeq[ExtendedSecretKey],
    usePreEip3Derivation: Boolean
): Try[DerivationPath] = {

  @tailrec
  def nextPath(accPath: List[Int], remaining: Seq[Seq[Int]]): Try[DerivationPath] = {
    if (!remaining.forall(_.isEmpty)) {
      val maxChildIdx = remaining.flatMap(_.headOption).max
      if (!Index.isHardened(maxChildIdx)) {
        Success(DerivationPath(0 +: (accPath :+ maxChildIdx + 1), publicBranch = false))
      } else {
        nextPath(accPath :+ maxChildIdx, remaining.map(_.drop(1)))
      }
    } else {
      Failure(new Exception("Out of non-hardened index space"))
    }
  }

  if (secrets.isEmpty || (secrets.size == 1 && secrets.head.path.isMaster)) {
    // First key after master
    val path = if (usePreEip3Derivation) {
      Constants.preEip3DerivationPath  // m/1
    } else {
      Constants.eip3DerivationPath     // m/44'/429'/0'/0/0
    }
    Success(path)
  } else {
    // For EIP-3: increase last segment (m/44'/429'/0'/0/0 → m/44'/429'/0'/0/1)
    // For old derivation: increase last non-hardened segment
    if (secrets.last.path.isEip3) {
      Success(secrets.last.path.increased)
    } else {
      nextPath(List.empty, secrets.map(_.path.decodedPath.tail))
    }
  }
}
```

### EIP-3 Path Helpers

```scala
// From JavaHelpers.scala:382-393
def eip3DerivationWithLastIndex(index: Int) = {
  val firstPath = Constants.eip3DerivationPath
  DerivationPath(firstPath.decodedPath.dropRight(1) :+ index, firstPath.publicBranch)
}

def eip3DerivationParent() = {
  val firstPath = Constants.eip3DerivationPath
  DerivationPath(firstPath.decodedPath.dropRight(1), firstPath.publicBranch)
}
```

## Serialization

### Derivation Path Serialization

```scala
// From DerivationPath.scala:133-147
object DerivationPathSerializer extends SigmaSerializer[DerivationPath, DerivationPath] {

  override def serialize(obj: DerivationPath, w: SigmaByteWriter): Unit = {
    w.put(if (obj.publicBranch) 0x01 else 0x00)
    w.putInt(obj.depth)
    obj.decodedPath.foreach(i => w.putBytes(Index.serializeIndex(i)))
  }

  override def parse(r: SigmaByteReader): DerivationPath = {
    val publicBranch = if (r.getByte() == 0x01) true else false
    val depth = r.getInt()
    val path = (0 until depth).map(_ => Index.parseIndex(r.getBytes(4)))
    DerivationPath(path, publicBranch)
  }
}
```

### Extended Secret Key Serialization

```scala
// From ExtendedSecretKey.scala:102-122
object ExtendedSecretKeySerializer extends SigmaSerializer[ExtendedSecretKey, ExtendedSecretKey] {

  override def serialize(obj: ExtendedSecretKey, w: SigmaByteWriter): Unit = {
    w.putBytes(obj.keyBytes)           // 32 bytes
    w.putBytes(obj.chainCode)          // 32 bytes
    val pathBytes = DerivationPathSerializer.toBytes(obj.path)
    w.putUInt(pathBytes.length)
    w.putBytes(pathBytes)
  }

  override def parse(r: SigmaByteReader): ExtendedSecretKey = {
    val keyBytes = r.getBytes(CryptoFacade.SecretKeyLength)
    val chainCode = r.getBytes(CryptoFacade.SecretKeyLength)
    val pathLen = r.getUInt().toIntExact
    val path = DerivationPathSerializer.fromBytes(r.getBytes(pathLen))
    new ExtendedSecretKey(keyBytes, chainCode, false, path)
  }
}
```

### Extended Public Key Serialization

```scala
// From ExtendedPublicKey.scala:63-85
object ExtendedPublicKeySerializer extends SigmaSerializer[ExtendedPublicKey, ExtendedPublicKey] {

  // 33 bytes: 1 byte sign + 32 bytes x-coordinate
  val PublicKeyBytesSize: Int = CryptoFacade.SecretKeyLength + 1

  override def serialize(obj: ExtendedPublicKey, w: SigmaByteWriter): Unit = {
    w.putBytes(obj.keyBytes)           // 33 bytes
    w.putBytes(obj.chainCode)          // 32 bytes
    val pathBytes = DerivationPathSerializer.toBytes(obj.path)
    w.putUInt(pathBytes.length)
    w.putBytes(pathBytes)
  }

  override def parse(r: SigmaByteReader): ExtendedPublicKey = {
    val keyBytes = r.getBytes(PublicKeyBytesSize)
    val chainCode = r.getBytes(CryptoFacade.SecretKeyLength)
    val pathLen = r.getUInt().toIntExact
    val path = DerivationPathSerializer.fromBytes(r.getBytes(pathLen))
    new ExtendedPublicKey(keyBytes, chainCode, path)
  }
}
```

## Issue #1627 Bug

Historical bug in key derivation:

```scala
// From ExtendedSecretKey.scala:68-76
val keyBytes = if (parentKey.usePre1627KeyDerivation) {
  // Bug: may be less than 32 bytes if childKey is small
  BigIntegers.asUnsignedByteArray(childKey)
} else {
  // Correct: always padded to 32 bytes
  BigIntegers.asUnsignedByteArray(CryptoFacade.SecretKeyLength, childKey)
}
```

**Problem**: BIP-32 requires 32-byte keys. Small BigIntegers produced fewer bytes.

**Impact**: Old wallets used incorrect derivation; new wallets must use `usePre1627KeyDerivation = false`.

## Usage Example

Complete key derivation workflow:

```scala
import org.ergoplatform.sdk.wallet.secrets._
import org.ergoplatform.sdk.wallet.Constants
import org.ergoplatform.sdk.JavaHelpers

// 1. From mnemonic to master key
val mnemonic = "abandon abandon abandon abandon abandon abandon " +
               "abandon abandon abandon abandon abandon about"
val masterKey = JavaHelpers.seedToMasterKey(
  SecretString.create(mnemonic),
  null,  // no passphrase
  usePre1627KeyDerivation = false
)

// 2. Derive EIP-3 keys
val eip3Path = Constants.eip3DerivationPath  // m/44'/429'/0'/0/0
val firstKey = masterKey.derive(eip3Path)
println(s"First address key: ${firstKey.publicImage}")

// 3. Derive subsequent addresses
val secondPath = eip3Path.increased  // m/44'/429'/0'/0/1
val secondKey = masterKey.derive(secondPath)

// 4. Get public extended key for watch-only wallet
val publicMaster = masterKey.publicKey
// Can derive non-hardened children only
val publicChild = publicMaster.child(0)

// 5. Custom path derivation
val customPath = DerivationPath.fromEncoded("m/44'/429'/1'/0/0").get
val accountOneKey = masterKey.derive(customPath)
```

### Watch-Only Wallet

```scala
// Export public key at m/44'/429'/0'/0 (parent of address keys)
val watchOnlyPath = JavaHelpers.eip3DerivationParent()
val watchOnlyKey = masterKey.derive(watchOnlyPath).publicKey

// Can derive address public keys without secrets
val addr0Pub = watchOnlyKey.child(0)  // m/44'/429'/0'/0/0 public
val addr1Pub = watchOnlyKey.child(1)  // m/44'/429'/0'/0/1 public
// Cannot derive hardened children
// watchOnlyKey.child(Index.hardIndex(0))  // FAILS!
```

## Security Considerations

### Hardened Derivation Protection

With non-hardened derivation, if an attacker obtains:
- A child private key, AND
- The parent chain code

They can compute the parent private key. Hardened derivation prevents this.

### Recommended Path Structure

```
m/44'/429'/account'/change/address
       │     │         │       │       └── Non-hardened (public derivation OK)
       │     │         │       └────────── Non-hardened (0=external, 1=internal)
       │     │         └────────────────── Hardened (account isolation)
       │     └──────────────────────────── Hardened (coin type isolation)
       └────────────────────────────────── Hardened (purpose isolation)
```

### Secret Zeroing

Always zero secrets when done:

```scala
val key = masterKey.derive(path)
try {
  // Use key...
} finally {
  key.zeroSecret()  // Overwrite keyBytes with zeros
}
```

## Exercises

1. **Implementation**: Write a function that derives all addresses for a given account (indices 0-19).

2. **Analysis**: Explain why revealing a non-hardened child private key and chain code compromises the parent private key.

3. **Design**: How would you implement multi-account support using EIP-3 paths?

4. **Security**: What is the minimum information needed for a watch-only wallet to track all addresses in an account?

## Summary

- **BIP-32** defines hierarchical deterministic key derivation
- **Derivation paths** use notation like `m/44'/429'/0'/0/0`
- **Hardened derivation** (`'`) requires private key; provides security isolation
- **Non-hardened derivation** allows public key derivation without secrets
- **EIP-3** standardizes Ergo's derivation path: `m/44'/429'/account'/change/address`
- **Extended keys** pair key material with chain code (64 bytes total)
- **Issue #1627** fixed key byte padding; old wallets need `usePre1627KeyDerivation = true`

## Further Reading

- Source: `sdk/shared/src/main/scala/org/ergoplatform/sdk/wallet/secrets/`
- BIP-32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- BIP-39: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
- BIP-44: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- EIP-3: https://github.com/ergoplatform/eips/blob/master/eip-0003.md

---
*[Previous: Chapter 27](./ch27-high-level-sdk.md) | [Next: Chapter 29](../part10/ch29-soft-fork-mechanism.md)*
