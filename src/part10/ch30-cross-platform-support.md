# Chapter 30: Cross-Platform Support

## Prerequisites

- **Required knowledge**: Elliptic Curve Cryptography (Chapter 9), Build systems
- **Related concepts**: SDK architecture (Chapter 27), Cryptographic operations
- **Prior chapters**: Chapter 29 (Soft-Fork Mechanism)

## Learning Objectives

- Understand Scala.js cross-compilation architecture
- Master the platform abstraction layer
- Learn crypto library differences between JVM and JS
- Understand performance considerations across platforms

## Source References

- `core/jvm/src/main/scala/sigma/crypto/Platform.scala`
- `core/js/src/main/scala/sigma/crypto/Platform.scala`
- `core/shared/src/main/scala/sigma/crypto/CryptoFacade.scala`
- `core/shared/src/main/scala/sigma/crypto/CryptoContext.scala`
- `core/jvm/src/main/scala/sigma/reflection/Platform.scala`
- `core/js/src/main/scala/sigma/reflection/Platform.scala`

## Introduction

SigmaState supports both JVM and JavaScript platforms through Scala.js cross-compilation. This enables:

1. **Node.js execution** - Server-side JavaScript environments
2. **Browser execution** - Client-side wallet applications
3. **Shared codebase** - Same logic across platforms
4. **Platform-specific optimizations** - Native crypto libraries where available

## Project Structure

```
sigmastate/
├── core/
│   ├── shared/      # Platform-independent code
│   │   └── src/main/scala/sigma/
│   │       ├── crypto/
│   │       │   ├── CryptoFacade.scala    # Abstraction layer
│   │       │   └── CryptoContext.scala   # Context interface
│   │       └── reflection/
│   │           └── RClass.scala          # Reflection abstraction
│   ├── jvm/         # JVM-specific code
│   │   └── src/main/scala/sigma/
│   │       ├── crypto/Platform.scala
│   │       └── reflection/Platform.scala
│   └── js/          # JavaScript-specific code
│       └── src/main/scala/sigma/
│           ├── crypto/Platform.scala
│           └── reflection/Platform.scala
```

## Platform Abstraction

### CryptoFacade

The central abstraction for cross-platform cryptography:

```scala
// From CryptoFacade.scala:9-151
object CryptoFacade {
  val Encoding = "UTF-8"
  val SecretKeyLength = 32
  val BitcoinSeed: Array[Byte] = "Bitcoin seed".getBytes(Encoding)
  val Pbkdf2Iterations = 2048
  val Pbkdf2KeyLength = 512

  /** Create a new cryptographic context */
  def createCryptoContext(): CryptoContext = Platform.createContext()

  /** Normalize point to affine coordinates */
  def normalizePoint(p: Ecp): Ecp = Platform.normalizePoint(p)

  /** Negate point (y-coordinate) */
  def negatePoint(p: Ecp): Ecp = Platform.negatePoint(p)

  /** Check if point is infinity */
  def isInfinityPoint(p: Ecp): Boolean = Platform.isInfinityPoint(p)

  /** Point exponentiation: p^n */
  def exponentiatePoint(p: Ecp, n: BigInteger): Ecp =
    Platform.exponentiatePoint(p, n)

  /** Point multiplication: p1 * p2 */
  def multiplyPoints(p1: Ecp, p2: Ecp): Ecp =
    Platform.multiplyPoints(p1, p2)

  /** ASN.1 encoding (compressed or uncompressed) */
  def getASN1Encoding(p: Ecp, compressed: Boolean): Array[Byte] =
    Platform.getASN1Encoding(p, compressed)

  /** Get X coordinate */
  def getXCoord(p: Ecp): ECFieldElem = Platform.getXCoord(p)

  /** Get Y coordinate */
  def getYCoord(p: Ecp): ECFieldElem = Platform.getYCoord(p)

  /** HMAC-SHA512 */
  def hashHmacSHA512(key: Array[Byte], data: Array[Byte]): Array[Byte] =
    Platform.hashHmacSHA512(key, data)

  /** PBKDF2 key derivation */
  def generatePbkdf2Key(normalizedMnemonic: String, normalizedPass: String): Array[Byte] =
    Platform.generatePbkdf2Key(normalizedMnemonic, normalizedPass)

  /** Unicode normalization (NFKD) */
  def normalizeChars(chars: Array[Char]): String = Platform.normalizeChars(chars)

  /** Secure random number generator */
  def createSecureRandom(): SecureRandom = Platform.createSecureRandom()
}
```

### CryptoContext

Abstract interface for elliptic curve operations:

```scala
// From CryptoContext.scala:6-37
abstract class CryptoContext {
  /** The underlying elliptic curve descriptor */
  def curve: Curve

  /** The characteristic of the underlying finite field */
  def fieldCharacteristic: BigInteger

  /** The order of the underlying group */
  def order: BigInteger

  /** Validate point coordinates */
  def validatePoint(x: BigInteger, y: BigInteger): Ecp

  /** Point at infinity */
  def infinity(): Ecp

  /** Decode point from bytes */
  def decodePoint(encoded: Array[Byte]): Ecp

  /** Group generator */
  def generator: Ecp
}

object CryptoContext {
  /** Default cryptographic context */
  val default: CryptoContext = Platform.createContext()
}
```

## JVM Platform Implementation

### Crypto Platform (JVM)

Uses Bouncy Castle library:

```scala
// From core/jvm/.../Platform.scala:18-210
object Platform {
  /** Wrapper for EC curve */
  case class Curve(private[crypto] val value: ECCurve)

  /** Wrapper for EC point */
  case class Ecp(private[crypto] val value: ECPoint)

  /** Wrapper for field element */
  case class ECFieldElem(value: ECFieldElement)

  /** Secure randomness */
  type SecureRandom = java.security.SecureRandom

  /** Create crypto context using secp256k1 */
  def createContext(): CryptoContext =
    new CryptoContextJvm(CustomNamedCurves.getByName("secp256k1"))

  def createSecureRandom(): SecureRandom = new SecureRandom()

  /** Point operations - BC uses additive notation */
  def multiplyPoints(p1: Ecp, p2: Ecp): Ecp = {
    // BC treats EC as additive group
    Ecp(p1.value.add(p2.value))
  }

  def exponentiatePoint(p: Ecp, n: BigInteger): Ecp = {
    // BC: exponentiate is multiply
    Ecp(p.value.multiply(n))
  }

  def isInfinityPoint(p: Ecp): Boolean = p.value.isInfinity

  def negatePoint(p: Ecp): Ecp = Ecp(p.value.negate())

  def normalizePoint(p: Ecp): Ecp = Ecp(p.value.normalize())

  def getASN1Encoding(p: Ecp, compressed: Boolean): Array[Byte] =
    p.value.getEncoded(compressed)

  /** Coordinate access */
  def getXCoord(p: Ecp): ECFieldElem = ECFieldElem(p.value.getXCoord)
  def getYCoord(p: Ecp): ECFieldElem = ECFieldElem(p.value.getYCoord)
  def getAffineXCoord(p: Ecp): ECFieldElem = ECFieldElem(p.value.getAffineXCoord)
  def getAffineYCoord(p: Ecp): ECFieldElem = ECFieldElem(p.value.getAffineYCoord)

  /** HMAC-SHA512 using Bouncy Castle */
  def hashHmacSHA512(key: Array[Byte], data: Array[Byte]): Array[Byte] =
    HmacSHA512.hash(key, data)

  /** PBKDF2 using Bouncy Castle */
  def generatePbkdf2Key(normalizedMnemonic: String, normalizedPass: String): Array[Byte] = {
    val gen = new PKCS5S2ParametersGenerator(new SHA512Digest)
    gen.init(
      normalizedMnemonic.getBytes(CryptoFacade.Encoding),
      normalizedPass.getBytes(CryptoFacade.Encoding),
      CryptoFacade.Pbkdf2Iterations)
    val dk = gen.generateDerivedParameters(CryptoFacade.Pbkdf2KeyLength)
            .asInstanceOf[KeyParameter].getKey
    dk
  }

  /** Unicode normalization using java.text.Normalizer */
  def normalizeChars(chars: Array[Char]): String = {
    Normalizer.normalize(ArrayCharSequence(chars), NFKD)
  }
}
```

### JVM-Specific Extensions

```scala
// From core/jvm/.../Platform.scala:203-208
implicit class EcpOps(val p: Ecp) extends AnyVal {
  def getCurve: ECCurve = p.value.getCurve
  def isInfinity: Boolean = CryptoFacade.isInfinityPoint(p)
  def add(p2: Ecp): Ecp = CryptoFacade.multiplyPoints(p, p2)
  def multiply(n: BigInteger): Ecp = CryptoFacade.exponentiatePoint(p, n)
}
```

## JavaScript Platform Implementation

### Crypto Platform (JS)

Uses `sigmajs-crypto-facade` npm package:

```scala
// From core/js/.../Platform.scala:15-269
object Platform {
  /** JS implementation of curve (placeholder) */
  class Curve

  /** Opaque JS point type */
  @js.native
  trait Point extends js.Object {
    def x: js.BigInt = js.native
    def y: js.BigInt = js.native
    def toHex(b: Boolean): String = js.native
  }

  /** JS EC point wrapper */
  class Ecp(val point: Point) {
    lazy val hex = point.toHex(true)
    override def hashCode(): Int = hex.hashCode
    override def equals(obj: Any): Boolean = (this eq obj.asInstanceOf[AnyRef]) ||
      (obj match {
        case that: Ecp => this.hex == that.hex
        case _ => false
      })
  }

  /** JS field element wrapper */
  class ECFieldElem(val elem: js.BigInt) {
    private lazy val digits: String = elem.toString(10)
    override def hashCode(): Int = digits.hashCode
    override def equals(obj: Any): Boolean = // similar to Ecp
  }

  /** Secure random - uses JS crypto */
  type SecureRandom = sigma.crypto.SecureRandomJS

  /** Byte array conversions */
  def Uint8ArrayToBytes(jsShorts: Uint8Array): Array[Byte] =
    jsShorts.toArray[Short].map(x => x.toByte)

  def bytesToJsShorts(bytes: Array[Byte]): js.Array[Short] =
    js.Array(bytes.map(x => (x & 0xFF).toShort): _*)

  /** Point operations via JS facade */
  def multiplyPoints(p1: Ecp, p2: Ecp): Ecp =
    new Ecp(CryptoFacadeJs.addPoint(p1.point, p2.point))

  def exponentiatePoint(p: Ecp, n: BigInteger): Ecp = {
    val sign = n.signum()
    if (sign == 0 || isInfinityPoint(p)) {
      val ctx = new CryptoContextJs()
      return new Ecp(ctx.getInfinity())
    }
    val scalar = Convert.bigIntegerToBigInt(n.abs())
    val positive = CryptoFacadeJs.multiplyPoint(p.point, scalar)
    val result = if (sign > 0) positive else CryptoFacadeJs.negatePoint(positive)
    new Ecp(result)
  }

  def isInfinityPoint(p: Ecp): Boolean =
    CryptoFacadeJs.isInfinityPoint(p.point)

  def getASN1Encoding(p: Ecp, compressed: Boolean): Array[Byte] = {
    val hex = if (isInfinityPoint(p)) "00"
              else p.point.toHex(compressed)
    Base16.decode(hex).get
  }

  /** Crypto via JS facade */
  def hashHmacSHA512(key: Array[Byte], data: Array[Byte]): Array[Byte] = {
    val keyArg = Uint8Array.from(bytesToJsShorts(key))
    val dataArg = Uint8Array.from(bytesToJsShorts(data))
    val hash = CryptoFacadeJs.hashHmacSHA512(keyArg, dataArg)
    Uint8ArrayToBytes(hash)
  }

  def generatePbkdf2Key(normalizedMnemonic: String, normalizedPass: String): Array[Byte] = {
    val res = CryptoFacadeJs.generatePbkdf2Key(normalizedMnemonic, normalizedPass)
    Uint8ArrayToBytes(res)
  }

  /** Unicode normalization via JS */
  def normalizeChars(chars: Array[Char]): String = {
    import js.JSStringOps._
    String.valueOf(chars).normalize(UnicodeNormalizationForm.NFKD)
  }
}
```

### BigInt Conversions

```scala
// From core/js/.../Platform.scala:170-180
object Convert {
  /** JavaScript BigInt → Java BigInteger */
  def bigIntToBigInteger(jsValue: js.BigInt): BigInteger =
    new BigInteger(jsValue.toString(10), 10)

  /** Java BigInteger → JavaScript BigInt */
  def bigIntegerToBigInt(value: BigInteger): js.BigInt =
    js.BigInt(value.toString(10))
}
```

### JS CryptoContext

```scala
// From core/js/.../Platform.scala:183-210
def createContext(): CryptoContext = new CryptoContext {
  private val ctx = new CryptoContextJs

  override def curve: Curve = ???  // Not implemented

  override def fieldCharacteristic: BigInteger =
    Convert.bigIntToBigInteger(ctx.getModulus())

  override def order: BigInteger =
    Convert.bigIntToBigInteger(ctx.getOrder())

  override def validatePoint(x: BigInteger, y: BigInteger): Ecp = {
    val point = ctx.validatePoint(
      Convert.bigIntegerToBigInt(x),
      Convert.bigIntegerToBigInt(y))
    new Ecp(point)
  }

  override def infinity(): Ecp = new Ecp(ctx.getInfinity())

  override def decodePoint(encoded: Array[Byte]): Ecp = {
    if (encoded(0) == 0) return infinity()
    new Ecp(ctx.decodePoint(Base16.encode(encoded)))
  }

  override def generator: Ecp = new Ecp(ctx.getGenerator())
}
```

## Reflection Platform

### JVM Reflection

Full Java reflection capabilities:

```scala
// From core/jvm/.../reflection/Platform.scala:7-88
object Platform {
  /** Thread-safe class storage */
  private val classes = TrieMap.empty[Class[_], JRClass[_]]
  val unknownClasses = TrieMap.empty[Class[_], JRClass[_]]

  /** Resolve class metadata */
  def resolveClass[T](clazz: Class[T]): RClass[T] = {
    val cls = memoize(classes)(clazz, new JRClass[T](clazz)).asInstanceOf[JRClass[T]]
    cls
  }

  /** Thread-safe cache */
  class Cache[K, V] {
    private val map: TrieMap[K, V] = TrieMap.empty[K, V]
    def getOrElseUpdate(key: K, value: => V): V = map.getOrElseUpdate(key, value)
  }

  /** Safe class name (workaround for Scala 2.11/2.12 bug) */
  def safeSimpleName(cl: Class[_]): String = {
    if (cl.getEnclosingClass == null) return cl.getSimpleName
    val simpleName = cl.getName.substring(cl.getEnclosingClass.getName.length)
    val length = simpleName.length
    var index = 0
    while (index < length && isSpecialChar(simpleName.charAt(index))) {
      index += 1
    }
    simpleName.substring(index)
  }

  def runtimePlatform: RuntimePlatform = RuntimePlatform.JVM
}
```

### JS Reflection

Pre-registered reflection data only:

```scala
// From core/js/.../reflection/Platform.scala:7-61
object Platform {
  /** Resolve from pre-registered data */
  def resolveClass[T](clazz: Class[T]): RClass[T] = {
    val res = ReflectionData.classes.get(clazz) match {
      case Some(c) =>
        assert(c.clazz == clazz)
        c
      case _ =>
        sys.error(s"Cannot find RClass data for $clazz")
    }
    res.asInstanceOf[RClass[T]]
  }

  /** Synchronized cache (HashMap-based) */
  class Cache[K, V] {
    private val map = mutable.HashMap.empty[K, V]
    def getOrElseUpdate(key: K, value: => V): V = synchronized {
      map.getOrElseUpdate(key, value)
    }
  }

  /** Scala 2.13+ safe name */
  def safeSimpleName(cl: Class[_]): String = cl.getSimpleName

  def runtimePlatform: RuntimePlatform = RuntimePlatform.JS
}
```

## Type Checking

### Platform-Specific Type Validation

```scala
// From core/jvm/.../Platform.scala:174-200
def isCorrectType[T <: SType](value: Any, tpe: T): Boolean = value match {
  case c: Coll[_] => tpe match {
    case STuple(items) => c.tItem == sigma.AnyType && c.length == items.length
    case _: SCollection[_] => true
    case _ => sys.error(s"Collection value $c has unexpected type $tpe")
  }
  case _: Option[_] => tpe.isOption
  case _: Tuple2[_, _] => tpe.isTuple && tpe.asTuple.items.length == 2
  case _: Boolean => tpe == SBoolean
  case _: Byte => tpe == SByte
  case _: Short => tpe == SShort
  case _: Int => tpe == SInt
  case _: Long => tpe == SLong
  case _: BigInt => tpe == SBigInt
  case _: UnsignedBigInt => tpe == SUnsignedBigInt
  case _: GroupElement => tpe.isGroupElement
  case _: SigmaProp => tpe.isSigmaProp
  case _: AvlTree => tpe.isAvlTree
  case _: Box => tpe.isBox
  case _: PreHeader => tpe == SPreHeader
  case _: Header => tpe == SHeader
  case _: Context => tpe == SContext
  case _: Function1[_, _] => tpe.isFunc
  case _: Unit => tpe == SUnit
  case _ => false
}
```

JS version has slight differences for numeric types:

```scala
// From core/js/.../Platform.scala:245-268
case _: Byte | _: Short | _: Int | _: Long => tpe.isInstanceOf[SNumericType]
// JS doesn't distinguish between numeric primitives at runtime
```

## Cryptographic Library Differences

### JVM: Bouncy Castle

- **Mature library** with extensive testing
- **Native performance** optimizations
- **Full EC operations** support
- **Secure random** via `java.security.SecureRandom`

### JS: sigmajs-crypto-facade

- **npm package** wrapping elliptic.js
- **Pure JavaScript** implementation
- **Cross-browser** compatibility
- **Web Crypto API** for randomness where available

## Performance Considerations

### JVM Advantages

1. **JIT compilation** - HotSpot optimization
2. **Native BigInteger** - Optimized arbitrary precision
3. **TrieMap** - Lock-free concurrent collections
4. **Bouncy Castle** - Battle-tested crypto

### JS Considerations

1. **Startup time** - Faster cold start
2. **Memory usage** - Lower footprint
3. **BigInt support** - Native in modern JS engines
4. **Async operations** - Better for I/O-bound work

### Optimization Strategies

```scala
// JVM: Use TrieMap for concurrent caching
private val classes = TrieMap.empty[Class[_], JRClass[_]]

// JS: Use synchronized HashMap
class Cache[K, V] {
  private val map = mutable.HashMap.empty[K, V]
  def getOrElseUpdate(key: K, value: => V): V = synchronized {
    map.getOrElseUpdate(key, value)
  }
}
```

## Build Configuration

### sbt Cross-Project Setup

```scala
// build.sbt (conceptual)
lazy val core = crossProject(JVMPlatform, JSPlatform)
  .crossType(CrossType.Full)
  .settings(commonSettings)
  .jvmSettings(
    libraryDependencies ++= Seq(
      "org.bouncycastle" % "bcprov-jdk18on" % bouncyCastleVersion
    )
  )
  .jsSettings(
    libraryDependencies ++= Seq(
      "org.scala-js" %%% "scalajs-dom" % scalajsDomVersion
    ),
    npmDependencies ++= Seq(
      "sigmajs-crypto-facade" -> "x.y.z"
    )
  )
```

## Usage Patterns

### Cross-Platform Code

```scala
import sigma.crypto.CryptoFacade

// Works on both JVM and JS
def generateKeyPair(): (Ecp, BigInteger) = {
  val ctx = CryptoFacade.createCryptoContext()
  val random = CryptoFacade.createSecureRandom()
  val privateKey = new BigInteger(256, random)
  val publicKey = CryptoFacade.exponentiatePoint(ctx.generator, privateKey)
  (publicKey, privateKey)
}
```

### Platform Detection

```scala
import sigma.reflection.Platform

def logPlatform(): Unit = {
  Platform.runtimePlatform match {
    case RuntimePlatform.JVM => println("Running on JVM")
    case RuntimePlatform.JS => println("Running on JavaScript")
  }
}
```

### Conditional Compilation

```scala
// In shared code
def someOperation(): Unit = {
  // This code compiles for both platforms
  val result = CryptoFacade.hashHmacSHA512(key, data)
}

// In jvm/ directory only
def jvmSpecificOperation(): Unit = {
  import Platform.EcpOps
  val point = someEcp
  point.getCurve  // JVM-specific extension method
}
```

## Testing Cross-Platform

### Shared Tests

Tests in `shared/` run on both platforms:

```scala
// In shared test directory
class CryptoFacadeSpec extends AnyFunSuite {
  test("point multiplication") {
    val ctx = CryptoFacade.createCryptoContext()
    val g = ctx.generator
    val twoG = CryptoFacade.multiplyPoints(g, g)
    val twoGAlt = CryptoFacade.exponentiatePoint(g, BigInteger.valueOf(2))
    assert(twoG == twoGAlt)
  }
}
```

### Platform-Specific Tests

```scala
// In jvm/ test directory
class BouncyCastleSpec extends AnyFunSuite {
  test("BC-specific operation") {
    // Test Bouncy Castle specific behavior
  }
}
```

## Exercises

1. **Implementation**: Add a new cross-platform method to `CryptoFacade` for EdDSA signatures.

2. **Analysis**: Compare the performance of point multiplication on JVM vs JS for 1000 operations.

3. **Design**: How would you add support for WebAssembly as a third platform?

4. **Debugging**: What challenges arise when debugging JS-compiled Scala code?

## Summary

- **Scala.js** enables JVM and JavaScript from single codebase
- **CryptoFacade** provides platform-agnostic crypto API
- **Platform object** contains platform-specific implementations
- **JVM uses Bouncy Castle**; JS uses sigmajs-crypto-facade
- **Reflection** is pre-registered on JS, dynamic on JVM
- **Type checking** differs slightly between platforms
- **Cache implementations** vary (TrieMap vs HashMap)

## Further Reading

- Source: `core/jvm/src/main/scala/sigma/crypto/Platform.scala`
- Source: `core/js/src/main/scala/sigma/crypto/Platform.scala`
- Scala.js Documentation: https://www.scala-js.org/
- Bouncy Castle: https://www.bouncycastle.org/
- sigmajs-crypto-facade: npm package

---
*[Previous: Chapter 29](./ch29-soft-fork-mechanism.md) | [Next: Chapter 31](./ch31-performance-engineering.md)*
