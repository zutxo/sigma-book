# Appendix F: Version History

This appendix provides the version history of ErgoScript and the SigmaState interpreter.

**Source**: `core/shared/src/main/scala/sigma/VersionContext.scala`, Language specification tests

## Protocol Versions Overview

| Block Version | Activated Version | ErgoTree Version | Name | Release |
|---------------|-------------------|------------------|------|---------|
| 1 | 0 | 0 | Initial | Mainnet launch |
| 2 | 1 | 1 | v4.0 | 2020 |
| 3 | 2 | 2 | v5.0 (JIT) | 2022 |
| 4 | 3 | 3 | v6.0 | 2024/2025 |

## Version Context

```scala
case class VersionContext(
  activatedVersion: Byte,  // Protocol version on network
  ergoTreeVersion: Byte    // Version of currently executing script
)

object VersionContext {
  val MaxSupportedScriptVersion: Byte = 3  // Supports 0, 1, 2, 3
  val JitActivationVersion: Byte = 2       // v5.0 JIT activation
  val V6SoftForkVersion: Byte = 3          // v6.0 soft-fork
}
```

## Version 1 (Initial - v3.x)

**ErgoTree Version**: 0

**Features**:
- Core ErgoScript language
- Basic types: Boolean, Byte, Short, Int, Long, BigInt, GroupElement, SigmaProp
- Collection operations: map, filter, fold, exists, forall
- Sigma protocols: ProveDlog, ProveDHTuple, AND, OR, THRESHOLD
- Box operations: value, propositionBytes, id, registers R0-R9
- Context access: INPUTS, OUTPUTS, HEIGHT, SELF

**Limitations**:
- AOT (Ahead-of-Time) interpreter only
- Fixed cost model
- No constant segregation required

## Version 2 (v4.0)

**ErgoTree Version**: 1
**Block Version**: 2

**New Features**:
- Mandatory constant segregation flag
- Improved script validation
- Enhanced soft-fork mechanism
- Size flag in ErgoTree header

**Changes**:
- ErgoTree header now requires size bytes when flag is set
- Better error handling for malformed scripts

## Version 3 (v5.0 - JIT)

**ErgoTree Version**: 2
**Block Version**: 3
**Activated Version**: 2

This was the major interpreter upgrade replacing AOT with JIT costing.

### Major Changes

**New Interpreter Architecture**:
- JIT (Just-In-Time) costing model
- Data-driven evaluation via `eval()` methods
- Precise cost tracking per operation
- Profiler support for cost measurement

**New Cost Model**:
- `FixedCost` for constant-time operations
- `PerItemCost` for collection operations
- `TypeBasedCost` for type-dependent costs
- `DynamicCost` for complex operations

**Costing Changes**:
```
AOT: Fixed costs estimated at compile time
JIT: Actual costs computed during execution
```

**New Operations**:
- `Context.dataInputs` - access data inputs
- `Context.headers` - access last 10 block headers
- `Context.preHeader` - access current block pre-header
- `Header` type with full block header access
- `PreHeader` type

**Soft-Fork Infrastructure**:
- `ValidationRules` framework
- Configurable rule status (enabled, disabled, replaced)
- `trySoftForkable` pattern for graceful degradation

### AOT to JIT Transition

The transition happened at a specific block height:

```scala
def isJitActivated: Boolean = activatedVersion >= JitActivationVersion
```

Scripts created before JIT activation continue to work, but new scripts benefit from more accurate costing.

## Version 4 (v6.0 - Evolution)

**ErgoTree Version**: 3
**Block Version**: 4
**Activated Version**: 3

This soft-fork adds significant new functionality.

### New Types

**SUnsignedBigInt** (Type code 9):
- 256-bit unsigned integers
- Modular arithmetic operations
- Conversion between signed/unsigned

### New Methods

**Numeric Types** (Byte, Short, Int, Long, BigInt):
- `toBytes`: Convert to byte array
- `toBits`: Convert to boolean array
- `bitwiseInverse`: Bitwise NOT
- `bitwiseOr`, `bitwiseAnd`, `bitwiseXor`: Bitwise operations
- `shiftLeft`, `shiftRight`: Bit shifting

**BigInt**:
- `toUnsigned`: Convert to UnsignedBigInt
- `toUnsignedMod`: Modular conversion

**UnsignedBigInt**:
- `modInverse`: Modular multiplicative inverse
- `plusMod`, `subtractMod`, `multiplyMod`: Modular arithmetic
- `mod`: Modulo operation
- `toSigned`: Convert to signed BigInt

**GroupElement**:
- `expUnsigned`: Scalar multiplication with unsigned exponent

**Header**:
- `checkPow`: Verify Proof-of-Work solution

**Collection**:
- `patch`: Replace range with another collection
- `updated`: Update single element
- `updateMany`: Batch update elements
- `indexOf`: Find element index
- `zip`: Pair with another collection
- `reverse`: Reverse order
- `startsWith`, `endsWith`: Prefix/suffix checks
- `get`: Safe element access returning Option

**Global**:
- `serialize`: Serialize any value to bytes
- `fromBigEndianBytes`: Decode big-endian bytes
- `encodeNBits`, `decodeNBits`: Difficulty encoding
- `powHit`: Autolykos2 PoW verification

### Version Checks in Code

```scala
// Check for v6 features
if (VersionContext.current.isV6Activated) {
  // Use new features
}

// Check ErgoTree version
if (VersionContext.current.isV3OrLaterErgoTreeVersion) {
  // Use v6 methods
}
```

## Backward Compatibility

### Script Compatibility

All scripts created for earlier versions continue to work:

1. **Version 0 scripts**: Execute with v0 semantics
2. **Version 1 scripts**: Execute with v1 semantics
3. **Version 2 scripts**: Execute with JIT costing
4. **Version 3 scripts**: Full v6 features available

### Method Resolution by Version

```scala
// Methods available depend on ErgoTree version
def methods: Seq[SMethod] = {
  if (VersionContext.current.isV3OrLaterErgoTreeVersion) {
    _v6Methods  // All methods including v6
  } else {
    _v5Methods  // Pre-v6 methods only
  }
}
```

### Soft-Fork Safety

Unknown opcodes and methods in future versions are handled gracefully:

```scala
ValidationRules.CheckValidOpCode(opCode, vs) match {
  case Validated => // Known opcode, proceed
  case SoftForkable => // Unknown but allowed, return default
  case Invalid => // Reject script
}
```

## Migration Guide

### For Script Authors

**v5 → v6**:
- Use `UnsignedBigInt` for modular arithmetic (more efficient)
- Use new collection methods (`reverse`, `zip`, etc.)
- Use `Header.checkPow` for PoW verification
- Use `Global.serialize` for value encoding

### For Node Operators

**Upgrading to v6**:
1. Update node software before activation height
2. No action needed for existing scripts
3. New features available after soft-fork activation

## Feature Matrix

| Feature | v3.x | v4.0 | v5.0 | v6.0 |
|---------|------|------|------|------|
| Basic types | ✓ | ✓ | ✓ | ✓ |
| Sigma protocols | ✓ | ✓ | ✓ | ✓ |
| JIT costing | - | - | ✓ | ✓ |
| Data inputs | - | - | ✓ | ✓ |
| Headers access | - | - | ✓ | ✓ |
| UnsignedBigInt | - | - | - | ✓ |
| Bitwise ops | - | - | - | ✓ |
| Collection updates | - | - | - | ✓ |
| PoW verification | - | - | - | ✓ |
| Serialization | - | - | - | ✓ |

## Test Coverage

Version-specific behavior is tested in:

- `LanguageSpecificationV5.scala` (~9,690 lines)
- `LanguageSpecificationV6.scala` (~3,081 lines)

These tests verify:
- All operations produce expected results
- Cost calculations are accurate
- Version-gated features work correctly
- Backward compatibility is maintained

---
*[Previous: Appendix E](./appendix-e-serialization.md) | [Back to Book](../SUMMARY.md)*
