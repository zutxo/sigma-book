# Appendix E: Serialization Format Reference

This appendix provides the reference for ErgoTree and value serialization formats.

**Source**: `docs/spec/serialization.tex`, `data/shared/src/main/scala/sigma/serialization/`

## Integer Encoding

### VLQ (Variable-Length Quantity)

Unsigned integers are encoded using VLQ with continuation bits:

```
Byte format: [C][D D D D D D D]
             |  |____________|
             |       |
             |       +-- 7 data bits
             +---------- Continuation bit (1 = more bytes follow)
```

**Encoding Rules**:
- If value < 128: single byte, continuation bit = 0
- Otherwise: 7 bits of value, continuation bit = 1, recurse with remaining bits

**Examples**:
| Value | Hex | Binary |
|-------|-----|--------|
| 0 | `00` | `00000000` |
| 127 | `7F` | `01111111` |
| 128 | `80 01` | `10000000 00000001` |
| 255 | `FF 01` | `11111111 00000001` |
| 300 | `AC 02` | `10101100 00000010` |

### ZigZag Encoding

Signed integers use ZigZag encoding before VLQ:

```
ZigZag(n) = (n << 1) ^ (n >> 31)  // for 32-bit
          = (n << 1) ^ (n >> 63)  // for 64-bit
```

**Mapping**:
| Signed | Encoded |
|--------|---------|
| 0 | 0 |
| -1 | 1 |
| 1 | 2 |
| -2 | 3 |
| 2 | 4 |
| ... | ... |

## Type Serialization

### Primitive Type Codes

| Type | Code (Decimal) | Code (Hex) |
|------|----------------|------------|
| `SBoolean` | 1 | 0x01 |
| `SByte` | 2 | 0x02 |
| `SShort` | 3 | 0x03 |
| `SInt` | 4 | 0x04 |
| `SLong` | 5 | 0x05 |
| `SBigInt` | 6 | 0x06 |
| `SGroupElement` | 7 | 0x07 |
| `SSigmaProp` | 8 | 0x08 |
| `SUnsignedBigInt` | 9 | 0x09 |

### Collection Types

Collections are encoded as base type + embedded element type:

```
Coll[T] code = 12 + (embeddable_code(T) * offset)
```

**Embeddable offset**: 12 (0x0C)

| Type | Code |
|------|------|
| `Coll[Byte]` | 12 + 2 = 14 |
| `Coll[Int]` | 12 + 4 = 16 |
| `Coll[Coll[Byte]]` | 24 + 2 = 26 |

### Non-Embeddable Types

| Type | Code (Decimal) | Code (Hex) |
|------|----------------|------------|
| `SBox` | 99 | 0x63 |
| `SAvlTree` | 100 | 0x64 |
| `SContext` | 101 | 0x65 |
| `SHeader` | 104 | 0x68 |
| `SPreHeader` | 105 | 0x69 |
| `SGlobal` | 106 | 0x6A |

### Generic Type Format

For types that don't fit the embeddable pattern:

```
[Type constructor code (1 byte)]
[Type arguments (each recursively serialized)]
```

## ErgoTree Format

### Header Byte

```
Bits: [V V V V][S][C][R][R]
      |______|  |  |  |__|
         |      |  |    |
         |      |  |    +-- Reserved (2 bits)
         |      |  +------- Constant segregation flag
         |      +---------- Size flag (has size bytes)
         +----------------- Version (4 bits, 0-15)
```

**Version Values**:
| Version | Protocol |
|---------|----------|
| 0 | v3.x (initial) |
| 1 | v4.x |
| 2 | v5.x (JIT) |
| 3 | v6.x |

### Complete ErgoTree Structure

```
[Header (1 byte)]
[Size (VLQ, optional if S=1)]
[Complexity (VLQ, optional)]
[Constants count (VLQ, if C=1)]
[Constants (each: type + value)]
[Root expression (serialized tree)]
```

### Constant Segregation

When the constant segregation flag is set:

1. All constants are collected and serialized first
2. In the expression tree, constants are replaced with `ConstantPlaceholder` nodes
3. Each placeholder contains an index into the constants array

**ConstantPlaceholder format**:
```
[Opcode: 0x73 (1 byte)]
[Index (VLQ)]
[Type (serialized type)]
```

## Value Serialization

### Primitive Values

| Type | Format |
|------|--------|
| `Boolean` | 0x00 (false) or 0x01 (true) |
| `Byte` | 1 byte signed |
| `Short` | ZigZag + VLQ |
| `Int` | ZigZag + VLQ |
| `Long` | ZigZag + VLQ |
| `BigInt` | [Length (1 byte)][Bytes (big-endian, 2's complement)] |

### GroupElement

33 bytes (SEC1 compressed point encoding):

```
[Prefix (1 byte): 0x02 or 0x03][X coordinate (32 bytes)]
```

Prefix indicates Y coordinate parity.

### SigmaProp

```
[Discriminator (1 byte)]
[Data (varies by discriminator)]
```

**Discriminators**:
| Value | Type | Data |
|-------|------|------|
| 0xCD | ProveDlog | GroupElement (33 bytes) |
| 0xCE | ProveDHTuple | 4 × GroupElement (132 bytes) |
| 0x98 | THRESHOLD | k, n, children... |
| 0x96 | AND | children... |
| 0x97 | OR | children... |

### Collections

```
[Element count (VLQ)]
[Element type (serialized type)]
[Elements (each serialized)]
```

**Special case for `Coll[Boolean]`**:
```
[Element count (VLQ)]
[Bits packed into bytes, LSB first]
```

### Box

```
[Value (VLQ Long)]
[ErgoTree (serialized)]
[Tokens count (VLQ)]
[Tokens: (token_id (32 bytes), amount (VLQ Long))*]
[Creation height (VLQ Int)]
[Registers R4-R9 (4 bits for presence flags, then values)]
[Transaction ID (32 bytes)]
[Output index (VLQ Short)]
```

## Expression Serialization

### General Format

```
[Opcode (1 byte)]
[Arguments (opcode-specific)]
```

### Common Patterns

**Unary operations**:
```
[Opcode]
[Operand (serialized expression)]
```

**Binary operations**:
```
[Opcode]
[Left operand (serialized expression)]
[Right operand (serialized expression)]
```

**Method calls**:
```
[Opcode: 0xDC]
[Type ID (1 byte)]
[Method ID (1 byte)]
[Receiver (serialized expression)]
[Arguments count (VLQ)]
[Arguments (each serialized)]
```

### Block Expressions

**ValDef** (value definition):
```
[Opcode: 0xD6]
[ID (VLQ)]
[Value type (optional, context-dependent)]
[Value expression]
```

**BlockValue**:
```
[Opcode: 0xD8]
[Item count (VLQ)]
[Items (ValDef or FunDef)*]
[Result expression]
```

**FuncValue** (lambda):
```
[Opcode: 0xD9]
[Argument count (VLQ)]
[Arguments (ID + type)*]
[Body expression]
```

## Proof Serialization

### Unchecked Tree

Sigma proofs are serialized as trees:

```
[Root node (recursive)]
```

**Node format** (varies by proposition type):
- **ProveDlog**: `[Commitment (33 bytes)][Response (32 bytes)]`
- **ProveDHTuple**: `[Commitment (33 bytes)][Response (32 bytes)]`
- **AND/OR**: `[Challenge (32 bytes)][Children*]`
- **THRESHOLD**: `[Challenge (32 bytes)][Children*][Polynomial coefficients]`

### Challenge Encoding

Challenges are 32-byte scalars computed via Fiat-Shamir transformation:
```
challenge = BLAKE2b256(message || commitments || public_inputs)
```

## Size Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Max ErgoTree size | 4KB | Serialized tree bytes |
| Max box size | 4KB | Total serialized box |
| Max constants | 255 | Per ErgoTree |
| Max registers | 10 | R0-R9 |
| Max tokens per box | 255 | Token types |
| Max BigInt bytes | 32 | 256 bits |

## Deserialization Validation

During deserialization, the following are validated:

1. **Opcode validity**: Must be known or soft-fork compatible
2. **Type consistency**: Types must match expected signatures
3. **Size limits**: Collections, strings, BigInts within bounds
4. **Structural validity**: Well-formed tree structure
5. **Version compatibility**: Features match declared version

---
*[Previous: Appendix D](./appendix-d-methods.md) | [Next: Appendix F](./appendix-f-version-history.md)*
