# Appendix C: Cost Table

This appendix provides the reference for all operation costs in the JIT cost model.

**Source**: `data/shared/src/main/scala/sigma/ast/CostKind.scala`, `values.scala`, `trees.scala`, `transformers.scala`

## Cost Kinds

### FixedCost
Operations with constant execution cost regardless of input size.

```scala
case class FixedCost(cost: JitCost) extends CostKind
```

### PerItemCost
Operations with cost proportional to collection size.

```scala
case class PerItemCost(baseCost: JitCost, perChunkCost: JitCost, chunkSize: Int)
```

**Formula**: `totalCost = baseCost + perChunkCost * ceil(nItems / chunkSize)`

### TypeBasedCost
Operations where cost depends on the type involved.

```scala
abstract class TypeBasedCost extends CostKind {
  def costFunc(tpe: SType): JitCost
}
```

### DynamicCost
Operations where cost is computed dynamically as sum of sub-operations.

## Fixed Cost Operations

| Operation | Cost | Description |
|-----------|------|-------------|
| `ConstantPlaceholder` | 1 | Reference to segregated constant |
| `Height` | 1 | Get current block height |
| `Inputs` | 1 | Get transaction inputs |
| `Outputs` | 1 | Get transaction outputs |
| `LastBlockUtxoRootHash` | 1 | Get UTXO root hash |
| `Self` | 1 | Get self box |
| `MinerPubkey` | 1 | Get miner public key |
| `ValUse` | 5 | Use a defined value |
| `TaggedVariable` | 5 | Get context variable |
| `SomeValue` | 5 | Create Option Some |
| `NoneValue` | 5 | Create Option None |
| `SelectField` | 8 | Select tuple field |
| `CreateProveDlog` | 10 | Create ProveDlog |
| `OptionGetOrElse` | 10 | Option.getOrElse |
| `OptionIsDefined` | 10 | Check Option defined |
| `OptionGet` | 10 | Option.get |
| `ExtractAmount` | 10 | Get box value |
| `ExtractScriptBytes` | 10 | Get proposition bytes |
| `ExtractId` | 10 | Get box ID |
| `Tuple` | 10 | Create tuple |
| `Select1` - `Select5` | 12 | Select tuple element |
| `ByIndex` | 14 | Collection access by index |
| `BoolToSigmaProp` | 15 | Bool to SigmaProp conversion |
| `DeserializeContext` | 15 | Deserialize from context |
| `DeserializeRegister` | 15 | Deserialize from register |
| `ByteArrayToLong` | 16 | Convert bytes to Long |
| `LongToByteArray` | 17 | Convert Long to bytes |
| `CreateProveDHTuple` | 20 | Create DHT tuple |
| `If` | 20 | Conditional expression |
| `LogicalNot` | 20 | Boolean NOT |
| `Negation` | 20 | Numeric negation |
| `ArithOp` (Plus, Minus, etc.) | 26 | Arithmetic operations |
| `ByteArrayToBigInt` | 30 | Convert bytes to BigInt |
| `SubstConstants` | 30 | Substitute constants |
| `SizeOf` | 30 | Collection size |
| `MultiplyGroup` | 40 | Group element multiplication |
| `ExtractRegisterAs` | 50 | Get register value |
| `Exponentiate` | 300 | BigInt exponentiation |
| `DecodePoint` | 900 | Decode group element |

## Per-Item Cost Operations

| Operation | Base | Per-Chunk | Chunk Size | Description |
|-----------|------|-----------|------------|-------------|
| `CalcBlake2b256` | 20 | 7 | 128 | Blake2b256 hash |
| `CalcSha256` | 20 | 8 | 64 | SHA256 hash |
| `MapCollection` | 20 | 1 | 10 | Map over collection |
| `Exists` | 20 | 5 | 10 | Exists predicate |
| `ForAll` | 20 | 5 | 10 | ForAll predicate |
| `Fold` | 20 | 1 | 10 | Fold collection |
| `Filter` | 20 | 5 | 10 | Filter collection |
| `FlatMap` | 20 | 5 | 10 | FlatMap collection |
| `Slice` | 10 | 2 | 100 | Slice collection |
| `Append` | 20 | 2 | 100 | Append to collection |
| `SigmaAnd` | 10 | 2 | 1 | Sigma AND |
| `SigmaOr` | 10 | 2 | 1 | Sigma OR |
| `AND` (logical) | 10 | 5 | 32 | Logical AND |
| `OR` (logical) | 10 | 5 | 32 | Logical OR |
| `XorOf` | 20 | 5 | 32 | XOR of collection |
| `AtLeast` | 20 | 3 | 1 | Threshold signature |
| `Xor` (bytes) | 10 | 2 | 128 | XOR byte arrays |

## Type-Based Costs

### Numeric Casting

| Target Type | Cost |
|-------------|------|
| `Byte`, `Short`, `Int`, `Long` | 10 |
| `BigInt` | 30 |
| `UnsignedBigInt` | 30 |

### Comparison Operations

Comparison cost depends on type complexity:

| Type | Cost |
|------|------|
| Primitive types | 10-20 |
| `BigInt` | 30 |
| Collections | PerItemCost |
| Tuples | Sum of component costs |

## Interpreter Overhead Costs

| Cost Type | Value | Description |
|-----------|-------|-------------|
| `interpreterInitCost` | 10,000 | One-time interpreter initialization |
| `inputCost` | 2,000 | Per transaction input |
| `dataInputCost` | 100 | Per data input |
| `outputCost` | 100 | Per transaction output |
| `tokenAccessCost` | 100 | Per token access |

## Cost Limits

| Parameter | Default Value | Description |
|-----------|---------------|-------------|
| `maxBlockCost` | 1,000,000 | Maximum cost per block |
| Script cost limit | ~JitCost(8,000,000) | Approximate single-script limit |

## Method Costs

Methods on types have their costs defined in the method descriptors. See Appendix D for method-specific costs.

### Box Methods

| Method | Cost | Description |
|--------|------|-------------|
| `value` | 10 | Get box value |
| `propositionBytes` | 10 | Get script bytes |
| `id` | 10 | Get box ID |
| `R0` - `R9` | 50 | Access register |
| `tokens` | 50 | Get tokens |
| `creationInfo` | 30 | Get creation info |

### GroupElement Methods

| Method | Cost | Description |
|--------|------|-------------|
| `getEncoded` | 30 | Get compressed encoding |
| `exp` | 300 | Exponentiation |
| `multiply` | 40 | Point multiplication |
| `negate` | 20 | Point negation |

### Collection Methods

| Method | Base | Per-Chunk | Chunk Size |
|--------|------|-----------|------------|
| `size` | 30 | - | - |
| `apply` | 14 | - | - |
| `map` | 20 | 1 | 10 |
| `filter` | 20 | 5 | 10 |
| `fold` | 20 | 1 | 10 |
| `exists` | 20 | 5 | 10 |
| `forall` | 20 | 5 | 10 |
| `flatMap` | 20 | 5 | 10 |
| `slice` | 10 | 2 | 100 |
| `append` | 20 | 2 | 100 |
| `indices` | 20 | 2 | 100 |

## Cost Calculation Example

For a transaction with:
- 2 inputs (each with proveDlog script)
- 1 data input
- 3 outputs

```
Base cost:
  interpreterInitCost  = 10,000
  inputCost × 2        =  4,000
  dataInputCost × 1    =    100
  outputCost × 3       =    300

Per-input script (proveDlog verification):
  CreateProveDlog      =     10
  BoolToSigmaProp      =     15
  Script overhead      = ~1,000
                       --------
  Per input            = ~1,025 × 2 = ~2,050

Total: ~16,450 JitCost units
```

---
*[Previous: Appendix B](./appendix-b-opcodes.md) | [Next: Appendix D](./appendix-d-methods.md)*
