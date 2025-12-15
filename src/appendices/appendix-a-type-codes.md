# Appendix A: Complete Type Code Table

This appendix provides the reference for all Type Codes used in ErgoTree serialization.

**Source**: `core/shared/src/main/scala/sigma/ast/SType.scala`

## Primitive Types

| Decimal | Hex | Type | Description |
|---------|-----|------|-------------|
| 1 | 0x01 | `SBoolean` | Boolean (true/false) |
| 2 | 0x02 | `SByte` | 8-bit signed integer |
| 3 | 0x03 | `SShort` | 16-bit signed integer |
| 4 | 0x04 | `SInt` | 32-bit signed integer |
| 5 | 0x05 | `SLong` | 64-bit signed integer |
| 6 | 0x06 | `SBigInt` | 256-bit signed integer |
| 7 | 0x07 | `SGroupElement` | Elliptic curve point (secp256k1) |
| 8 | 0x08 | `SSigmaProp` | Sigma proposition (proof of knowledge) |
| 9 | 0x09 | `SUnsignedBigInt` | 256-bit unsigned integer (v6+ only) |
| 97 | 0x61 | `SAny` | Supertype of all types |
| 98 | 0x62 | `SUnit` | Unit type (singleton value) |
| 99 | 0x63 | `SBox` | Transaction box |
| 100 | 0x64 | `SAvlTree` | Authenticated dictionary (AVL+) |
| 101 | 0x65 | `SContext` | Execution context |
| 102 | 0x66 | `SString` | String (ErgoScript only, not in ErgoTree) |
| 103 | 0x67 | `STypeVar` | Type variable (Internal) |
| 104 | 0x68 | `SHeader` | Block header |
| 105 | 0x69 | `SPreHeader` | Block pre-header |
| 106 | 0x6A | `SGlobal` | Global object (SigmaDslBuilder) |

## Type Constructors

These codes are used for constructing complex types.

| Decimal | Hex | Type Constructor | Description |
|---------|-----|------------------|-------------|
| 12 | 0x0C | `SCollectionType` | `Coll[T]` (Generic collection) |
| 24 | 0x18 | `NestedCollectionType` | `Coll[Coll[T]]` |
| 36 | 0x24 | `SOption` | `Option[T]` |
| 60 | 0x3C | `Pair1Type` | `(_, T)` where first element is unknown/generic |
| 72 | 0x48 | `Pair2Type` | `(T, _)` where second element is unknown/generic |
| 84 | 0x54 | `PairSymmetricType` | `(T, T)` symmetric pair |
| 96 | 0x60 | `STuple` | Generic Tuple `(T1, ..., Tn)` |

> **Note**: Type codes 1-11 are reserved for primitive types. `LastDataType` is 111.

---
*[Previous: Chapter 31](../part10/ch31-performance-engineering.md) | [Next: Appendix B](./appendix-b-opcodes.md)*
