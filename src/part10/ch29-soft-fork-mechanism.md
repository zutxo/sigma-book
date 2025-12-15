# Chapter 29: Soft-Fork Mechanism

## Prerequisites

- **Required knowledge**: ErgoTree structure (Chapter 3), Serialization (Chapter 7)
- **Related concepts**: Transaction validation (Chapter 24), Cost model (Chapter 13)
- **Prior chapters**: Chapter 28 (Key Derivation)

## Learning Objectives

- Understand version context and script versioning
- Master validation rule status and soft-fork conditions
- Learn unknown opcode handling
- Understand the AOT to JIT interpreter transition

## Source References

- `core/shared/src/main/scala/sigma/VersionContext.scala`
- `core/shared/src/main/scala/sigma/validation/ValidationRules.scala`
- `core/shared/src/main/scala/sigma/validation/SigmaValidationSettings.scala`
- `core/shared/src/main/scala/sigma/validation/SoftForkChecker.scala`
- `core/shared/src/main/scala/sigma/validation/RuleStatus.scala`
- `docs/aot-jit-switch.md`

## Introduction

Ergo's soft-fork mechanism enables protocol upgrades without breaking consensus among nodes running different versions. Key features include:

1. **Version context** - Track activated protocol and ErgoTree versions
2. **Validation rules** - Configurable rules with statuses that can change via voting
3. **Soft-fork detection** - Recognize new opcodes/types as soft-fork conditions
4. **Graceful degradation** - Old nodes can accept blocks they cannot fully verify

## Version Context

The `VersionContext` tracks the current protocol and script versions:

```scala
// From VersionContext.scala:17-35
case class VersionContext(activatedVersion: Byte, ergoTreeVersion: Byte) {
  // Validate: ergoTreeVersion <= activatedVersion
  require(activatedVersion < JitActivationVersion || ergoTreeVersion <= activatedVersion,
    s"ergoTreeVersion must never exceed activatedVersion: $this")

  /** True if JIT costing is activated (v5.0+) */
  def isJitActivated: Boolean = activatedVersion >= JitActivationVersion

  /** True if v6.0 ErgoTree version */
  def isV3OrLaterErgoTreeVersion: Boolean = ergoTreeVersion >= V6SoftForkVersion

  /** True if v6.0 protocol is activated */
  def isV6Activated: Boolean = activatedVersion >= V6SoftForkVersion
}
```

### Version Constants

```scala
// From VersionContext.scala:47-56
object VersionContext {
  /** Maximum supported ErgoTree version (0, 1, 2, 3) */
  val MaxSupportedScriptVersion: Byte = 3

  /** JIT costing activation version (v5.0) */
  val JitActivationVersion: Byte = 2

  /** v6.0 soft-fork version */
  val V6SoftForkVersion: Byte = 3
}
```

### Version History

| Block Version | Script Version | Protocol | Features |
|---------------|----------------|----------|----------|
| 1 | 0 | v3.x | Initial mainnet |
| 2 | 1 | v4.x | Height monotonicity |
| 3 | 2 | v5.x | JIT interpreter |
| 4 | 3 | v6.x | Sub-blocks, new operations |

### Thread-Local Context

Version context is thread-local via `DynamicVariable`:

```scala
// From VersionContext.scala:68-100
object VersionContext {
  private val _versionContext: DynamicVariable[VersionContext] =
    new DynamicVariable[VersionContext](_defaultContext)

  /** Get current version context for this thread */
  def current: VersionContext = {
    val ctx = _versionContext.value
    if (ctx == null)
      throw new IllegalStateException("VersionContext not specified")
    ctx
  }

  /** Execute block with specific version context */
  def withVersions[T](activatedVersion: Byte, ergoTreeVersion: Byte)(block: => T): T =
    _versionContext.withValue(VersionContext(activatedVersion, ergoTreeVersion))(block)

  /** Verify current context matches expected versions */
  def checkVersions(activatedVersion: Byte, ergoTreeVersion: Byte): Unit = {
    val ctx = VersionContext.current
    if (ctx.activatedVersion != activatedVersion || ctx.ergoTreeVersion != ergoTreeVersion) {
      throw new IllegalStateException(s"Expected $expected but got $ctx")
    }
  }
}
```

## Rule Status

Validation rules can have different statuses:

```scala
// From RuleStatus.scala:4-53
sealed trait RuleStatus {
  def statusCode: Byte
}

object RuleStatus {
  val EnabledRuleCode: Byte = 1
  val DisabledRuleCode: Byte = 2
  val ReplacedRuleCode: Byte = 3
  val ChangedRuleCode: Byte = 4
}

/** Default: rule is active and enforced */
case object EnabledRule extends RuleStatus {
  val statusCode: Byte = RuleStatus.EnabledRuleCode
}

/** Rule is disabled (via voting) */
case object DisabledRule extends RuleStatus {
  val statusCode: Byte = RuleStatus.DisabledRuleCode
}

/** Rule replaced by new rule via soft-fork
  * @param newRuleId ID of the replacement rule
  */
case class ReplacedRule(newRuleId: Short) extends RuleStatus {
  val statusCode: Byte = RuleStatus.ReplacedRuleCode
}

/** Rule parameters changed via soft-fork
  * @param newValue new configuration data
  */
case class ChangedRule(newValue: Array[Byte]) extends RuleStatus {
  val statusCode: Byte = RuleStatus.ChangedRuleCode
}
```

## Validation Rules

### ValidationRule Base Class

```scala
// From ValidationRules.scala:13-51
abstract case class ValidationRule(
    id: Short,
    description: String
) extends SoftForkChecker {
  protected def settings: SigmaValidationSettings
  private var _checked: Boolean = false

  /** Check rule is registered (HOTSPOT - only checked once) */
  @inline protected final def checkRule(): Unit = {
    if (!_checked) {
      if (settings.getStatus(this.id).isEmpty) {
        throw new SigmaException(s"ValidationRule $this not found")
      }
      _checked = true
    }
  }

  /** Throw ValidationException with cause and args */
  def throwValidationException(cause: Throwable, args: Seq[Any]): Nothing = {
    if (cause.isInstanceOf[ValidationException]) {
      throw cause
    } else {
      throw ValidationException(
        s"Validation failed on $this with args $args",
        this, args, Option(cause))
    }
  }
}
```

### ValidationException

```scala
// From ValidationRules.scala:65-68
case class ValidationException(
    message: String,
    rule: ValidationRule,
    args: Seq[Any],
    cause: Option[Throwable] = None
) extends Exception(message, cause.orNull) {
  // Skip stack trace for performance
  override def fillInStackTrace(): Throwable = this
}
```

### Core Validation Rules

```scala
// From ValidationRules.scala:80-192
object ValidationRules {
  val FirstRuleId = 1000.toShort

  /** Check primitive type code is valid */
  object CheckPrimitiveTypeCode extends ValidationRule(1007,
    "Check the primitive type code is supported or added via soft-fork")
      with SoftForkWhenCodeAdded {
    final def apply[T](code: Byte): Unit = {
      checkRule()
      val ucode = toUByte(code)
      if (ucode <= 0 || ucode >= embeddableIdToType.length) {
        throwValidationException(
          new SerializerException(s"Cannot deserialize type with code $ucode"),
          Array(code))
      }
    }
  }

  /** Check non-primitive type code is valid */
  object CheckTypeCode extends ValidationRule(1008,
    "Check non-primitive type code is supported or added via soft-fork")
      with SoftForkWhenCodeAdded {
    final def apply[T](typeCode: Byte): Unit = {
      checkRule()
      val ucode = toUByte(typeCode)
      if (ucode > toUByte(SGlobal.typeCode)) {
        throwValidationException(
          new SerializerException(s"Cannot deserialize type with code $ucode"),
          Array(typeCode))
      }
    }
  }

  /** Check data can be serialized for type */
  object CheckSerializableTypeCode extends ValidationRule(1009,
    "Check data values of type can be serialized") with SoftForkWhenReplaced {
    final def apply[T](typeCode: Byte): Unit = {
      checkRule()
      val ucode = toUByte(typeCode)
      if (typeCode == SOption.OptionTypeCode || ucode > toUByte(TypeCodes.LastDataType)) {
        throwValidationException(typeCode)
      }
    }
  }

  /** Check reader hasn't exceeded position limit */
  object CheckPositionLimit extends ValidationRule(1014,
    "Check Reader position limit") with SoftForkWhenReplaced {
    final def apply(position: Int, positionLimit: Int): Unit = {
      checkRule()
      if (position > positionLimit) {
        throwValidationException(position, positionLimit)
      }
    }
  }
}
```

### Version-Specific Rules

Different rule sets for different protocol versions:

```scala
// From ValidationRules.scala:194-234
private val ruleSpecsV5: Seq[ValidationRule] = Seq(
  CheckPrimitiveTypeCode,
  CheckTypeCode,
  CheckSerializableTypeCode,
  CheckTypeWithMethods,
  CheckPositionLimit
)

private val ruleSpecsV6: Seq[ValidationRule] = Seq(
  CheckPrimitiveTypeCodeV6,  // Updated for v6
  CheckTypeCodeV6,           // Updated for v6
  CheckSerializableTypeCode,
  CheckTypeWithMethods,
  CheckPositionLimit
)

def coreSettings: SigmaValidationSettings = {
  if (VersionContext.current.isV6Activated) {
    coreSettingsV6
  } else {
    coreSettingsV5
  }
}
```

## Soft-Fork Checkers

### SoftForkChecker Trait

```scala
// From SoftForkChecker.scala:4-13
trait SoftForkChecker {
  /** Check if condition represents a soft-fork
    * @param vs      Validation settings from blockchain
    * @param ruleId  ID of rule that raised exception
    * @param status  Rule status from blockchain
    * @param args    Arguments that caused exception
    * @return true if this is a valid soft-fork condition
    */
  def isSoftFork(vs: SigmaValidationSettings, ruleId: Short,
                 status: RuleStatus, args: Seq[Any]): Boolean = false
}
```

### SoftForkWhenReplaced

For rules that get replaced by new rules:

```scala
// From SoftForkChecker.scala:19-27
trait SoftForkWhenReplaced extends SoftForkChecker {
  override def isSoftFork(vs: SigmaValidationSettings,
      ruleId: Short, status: RuleStatus, args: Seq[Any]): Boolean =
    (status, args) match {
      case (ReplacedRule(_), _) => true
      case _ => false
    }
}
```

### SoftForkWhenCodeAdded

For new opcodes/types added via soft-fork:

```scala
// From SoftForkChecker.scala:34-42
trait SoftForkWhenCodeAdded extends SoftForkChecker {
  override def isSoftFork(vs: SigmaValidationSettings,
      ruleId: Short, status: RuleStatus, args: Seq[Any]): Boolean =
    (status, args) match {
      case (ChangedRule(newValue), Seq(code: Byte)) =>
        newValue.contains(code)  // Code explicitly added to blockchain
      case _ => false
    }
}
```

## Validation Settings

### SigmaValidationSettings

```scala
// From SigmaValidationSettings.scala:45-69
abstract class SigmaValidationSettings
    extends Iterable[(Short, (ValidationRule, RuleStatus))] {

  def get(id: Short): Option[(ValidationRule, RuleStatus)]
  def getStatus(id: Short): Option[RuleStatus]
  def updated(id: Short, newStatus: RuleStatus): SigmaValidationSettings

  /** Check if exception represents a soft-fork condition */
  def isSoftFork(ve: ValidationException): Boolean = {
    val ruleId = ve.rule.id
    val infoOpt = get(ruleId)
    infoOpt match {
      // Don't tolerate replaced v5.0 rules after v6.0 activation
      case Some((vr, ReplacedRule(_))) =>
        if ((vr.id == 1011 || vr.id == 1007 || vr.id == 1008) &&
            VersionContext.current.isV6Activated) {
          false
        } else {
          true
        }
      case Some((rule, status)) =>
        rule.isSoftFork(this, rule.id, status, ve.args)
      case None => false
    }
  }
}
```

### MapSigmaValidationSettings

```scala
// From SigmaValidationSettings.scala:72-94
sealed class MapSigmaValidationSettings(
    private val map: Map[Short, (ValidationRule, RuleStatus)]
) extends SigmaValidationSettings {

  override def iterator: Iterator[(Short, (ValidationRule, RuleStatus))] =
    map.iterator

  override def get(id: Short): Option[(ValidationRule, RuleStatus)] =
    map.get(id)

  /** HOTSPOT: optimized status lookup */
  override def getStatus(id: Short): Option[RuleStatus] = {
    val statusOpt = map.get(id)
    if (statusOpt.isDefined) Some(statusOpt.get._2) else None
  }

  override def updated(id: Short, newStatus: RuleStatus): MapSigmaValidationSettings = {
    val (rule, _) = map(id)
    new MapSigmaValidationSettings(map.updated(id, (rule, newStatus)))
  }
}
```

## Soft-Fork Handling

### trySoftForkable

Execute code with soft-fork fallback:

```scala
// From ValidationRules.scala:248-256
def trySoftForkable[T](whenSoftFork: => T)(block: => T)
                      (implicit vs: SigmaValidationSettings): T = {
  try block
  catch {
    case ve: ValidationException =>
      if (vs.isSoftFork(ve)) whenSoftFork
      else throw ve
  }
}
```

Usage example:

```scala
// When deserializing unknown opcode
trySoftForkable(whenSoftFork = {
  // Return placeholder for unknown operation
  ConstantPlaceholder(0, SUnit)
}) {
  // Normal deserialization
  ValueSerializer.deserialize(reader)
}
```

## AOT to JIT Transition

The v5.0 soft-fork transitioned from AOT (Ahead-Of-Time) to JIT (Just-In-Time) costing.

### Script Validation Rules

```
Rule | BlockVer | Block Type | Script Ver | v4.0 Action      | v5.0 Action
-----|----------|------------|------------|------------------|------------------
1    | 1,2      | candidate  | v0/v1      | AOT-cost,verify  | AOT-cost,verify
2    | 1,2      | mined      | v0/v1      | AOT-cost,verify  | AOT-cost,verify
3    | 1,2      | candidate  | v2         | skip-pool-tx     | skip-pool-tx
4    | 1,2      | mined      | v2         | skip-reject      | skip-reject
-----|----------|------------|------------|------------------|------------------
5    | 3        | candidate  | v0/v1      | skip-pool-tx     | JIT-verify
6    | 3        | mined      | v0/v1      | skip-accept      | JIT-verify
7    | 3        | candidate  | v2         | skip-pool-tx     | JIT-verify
8    | 3        | mined      | v2         | skip-accept      | JIT-verify
```

### Validation Actions

| Action | Description |
|--------|-------------|
| AOT-cost,verify | Cost estimation and verification using v4.0 AOT interpreter |
| JIT-verify | Verification using v5.0 JIT interpreter with just-in-time costing |
| skip-pool-tx | Skip mempool transaction (can't handle) |
| skip-accept | Accept block without verification (rely on majority) |
| skip-reject | Reject transaction/block (invalid version) |

### Equivalence Properties

For consensus compatibility:

1. **Prop 1**: AOT-verify preserved between releases
   ```
   forall s:ScriptV0/V1, R4.0-AOT-verify(s) == R5.0-AOT-verify(s)
   ```

2. **Prop 2**: AOT-cost preserved
   ```
   forall s:ScriptV0/V1, R4.0-AOT-cost(s) == R5.0-AOT-cost(s)
   ```

3. **Prop 3**: JIT can replace AOT
   ```
   forall s:ScriptV0/V1, R5.0-JIT-verify(s) == R4.0-AOT-verify(s)
   ```

4. **Prop 4**: JIT cost bounded by AOT
   ```
   forall s:ScriptV0/V1, R5.0-JIT-cost(s) <= R4.0-AOT-cost(s)
   ```

5. **Prop 5**: ScriptV2 rejected before SF
   ```
   forall s:ScriptV2, if not SF active => reject
   ```

## Version Context Usage

### In Interpreter

```scala
def verify(ergoTree: ErgoTree, ctx: ErgoLikeContext): Boolean = {
  val scriptVersion = ergoTree.version
  val activatedVersion = ctx.activatedScriptVersion

  VersionContext.withVersions(activatedVersion, scriptVersion) {
    // Execute with proper version context
    val reduced = fullReduction(ergoTree, ctx, env)
    verifySignature(reduced, ctx.messageToSign)
  }
}
```

### In Serialization

```scala
def deserializeErgoTree(bytes: Array[Byte]): ErgoTree = {
  val header = bytes(0)
  val treeVersion = ErgoTree.getVersion(header)

  VersionContext.withVersions(
    VersionContext.current.activatedVersion,
    treeVersion
  ) {
    // Deserialize with tree's version context
    ErgoTreeSerializer.deserialize(bytes)
  }
}
```

### Version Checks

```scala
// Only available in v6.0+
def someV6Operation(): Value[SType] = {
  if (!VersionContext.current.isV6Activated) {
    throw new InterpreterException("Operation requires v6.0")
  }
  // ... implementation
}
```

## Block Extension Voting

Rule statuses can be changed via blockchain extension sections:

```
Extension Key    | Extension Value
-----------------|--------------------
Rule ID (2 bytes)| RuleStatus + data
```

### Status Update Flow

1. Miners include votes in block extensions
2. Votes accumulate over voting epoch
3. When threshold reached, status changes
4. New status applies to subsequent blocks

### Example: Adding New Opcode

1. **Before soft-fork**: Unknown opcode throws `ValidationException`
2. **Extension update**: `ChangedRule(Array(newOpcode))` for rule 1001
3. **After activation**: Old nodes recognize soft-fork condition via `SoftForkWhenCodeAdded`
4. **Result**: Old nodes skip verification; new nodes execute new opcode

## Practical Implications

### For Node Operators

- **Old nodes** can stay in sync by trusting majority
- **Upgrade recommended** for full validation
- **No emergency** - network remains functional

### For Developers

- Use `VersionContext.current` to check available features
- Wrap new operations in version checks
- Test against multiple protocol versions

### For Scripts

```
// Script version in ErgoTree header
Version | Features
--------|----------
0       | Initial (v3.x)
1       | Height monotonicity (v4.x)
2       | JIT interpreter (v5.x)
3       | Sub-blocks, new ops (v6.x)
```

## Code Example

Handling unknown opcodes with soft-fork:

```scala
import sigma.validation.ValidationRules._
import sigma.validation.{SigmaValidationSettings, ValidationException}

def deserializeValue(reader: SigmaByteReader)
                    (implicit vs: SigmaValidationSettings): Value[SType] = {
  val opCode = reader.getByte()

  trySoftForkable(
    whenSoftFork = {
      // Unknown opcode - return placeholder for old nodes
      reader.position = reader.positionLimit  // Skip remaining bytes
      UnitConstant  // Treat as unit (will fail signature check safely)
    }
  ) {
    // Normal deserialization
    val serializer = getSerializer(opCode)
    serializer.parse(reader)
  }
}
```

## Exercises

1. **Analysis**: Trace through what happens when a v4.0 node encounters a v2 script in a mined block after soft-fork activation.

2. **Design**: How would you add a new primitive type via soft-fork? What rule changes are needed?

3. **Implementation**: Write a version-aware operation that behaves differently in v5.0 vs v6.0.

4. **Security**: Why is it important that old nodes can't be tricked into accepting invalid blocks via soft-fork?

## Summary

- **VersionContext** tracks activated protocol and ErgoTree versions per thread
- **ValidationRules** define configurable validation with soft-fork support
- **RuleStatus** can be `Enabled`, `Disabled`, `Replaced`, or `Changed`
- **SoftForkChecker** traits detect soft-fork conditions from exceptions
- **trySoftForkable** provides graceful fallback for unknown constructs
- **AOT→JIT transition** demonstrated soft-fork for major interpreter change
- **Old nodes** remain compatible by trusting majority on blocks they can't verify

## Further Reading

- Source: `core/shared/src/main/scala/sigma/VersionContext.scala`
- Source: `core/shared/src/main/scala/sigma/validation/`
- Documentation: `docs/aot-jit-switch.md`
- EIP-37 (Sub-blocks): https://github.com/ergoplatform/eips/pull/37

---
*[Previous: Chapter 28](../part9/ch28-key-derivation.md) | [Next: Chapter 30](./ch30-cross-platform-support.md)*
