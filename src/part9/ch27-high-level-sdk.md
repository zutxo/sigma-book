# Chapter 27: High-Level SDK

## Prerequisites

- **Required knowledge**: ErgoTree structure (Chapter 3), Transaction validation (Chapter 24)
- **Related concepts**: Prover implementation (Chapter 15), Cost model (Chapter 13)
- **Prior chapters**: Chapter 26 (Wallet and Signing)

## Learning Objectives

- Understand the high-level SDK architecture
- Master transaction building with the builder pattern
- Learn the SigmaProver interface for signing
- Understand the reduction pipeline
- Work with JSON codecs for API integration

## Source References

- `sdk/shared/src/main/scala/org/ergoplatform/sdk/SigmaProver.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/ProverBuilder.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/UnsignedTransactionBuilder.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/ReducingInterpreter.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/AppkitProvingInterpreter.scala`
- `sdk/shared/src/main/scala/org/ergoplatform/sdk/JsonCodecs.scala`

## Introduction

The Sigma SDK provides high-level APIs for building and signing Ergo transactions. It abstracts away the complexity of the interpreter and cryptographic protocols, offering:

1. **Builder pattern** for constructing transactions
2. **SigmaProver** for signing transactions
3. **ReducingInterpreter** for script reduction without signing
4. **JSON codecs** for API serialization

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       Application Layer                          │
├─────────────────────────────────────────────────────────────────┤
│  UnsignedTransactionBuilder    ProverBuilder    JsonCodecs      │
├─────────────────────────────────────────────────────────────────┤
│           SigmaProver              ReducingInterpreter           │
├─────────────────────────────────────────────────────────────────┤
│              AppkitProvingInterpreter                            │
├─────────────────────────────────────────────────────────────────┤
│           ProverInterpreter    ReducingInterpreter               │
├─────────────────────────────────────────────────────────────────┤
│                    ErgoLikeInterpreter                           │
└─────────────────────────────────────────────────────────────────┘
```

## Blockchain Context

The SDK operates within a blockchain context that provides network state:

```scala
// From BlockchainContext.scala:14-22
case class BlockchainContext(
    networkType: NetworkType,
    parameters: BlockchainParameters,
    stateContext: BlockchainStateContext
) {
  def headers: Coll[Header] = stateContext.sigmaLastHeaders
  def height: Int = headers(0).height
}
```

### Blockchain Parameters

Parameters control transaction validation costs:

```scala
// From BlockchainParameters.scala:6-29
abstract class BlockchainParameters {
  def storageFeeFactor: Int      // Storage cost per byte for 4 years
  def minValuePerByte: Int       // Minimum nanoErgs per byte
  def maxBlockSize: Int          // Maximum block size in bytes
  def tokenAccessCost: Int       // Cost per token access
  def inputCost: Int             // Cost per input (2000 default)
  def dataInputCost: Int         // Cost per data input (100 default)
  def outputCost: Int            // Cost per output (100 default)
  def maxBlockCost: Int          // Block computation limit
  def blockVersion: Byte         // Protocol version
}
```

### SDK Constants

```scala
// From BlockchainParameters.scala:47-65
object BlockchainParameters {
  val MinerRewardDelay_Mainnet = 720  // Blocks before reward spendable
  val MinerRewardDelay_Testnet = 720

  val OneErg: Long = 1000 * 1000 * 1000       // 10^9 nanoErgs
  val MinFee: Long = 1000 * 1000              // 0.001 ERG minimum
  val MinChangeValue: Long = 1000 * 1000      // Below this, change → fee
}
```

## Transaction Building

### UnsignedTransactionBuilder

The builder pattern for constructing transactions:

```scala
// From UnsignedTransactionBuilder.scala:20-28
class UnsignedTransactionBuilder(val ctx: BlockchainContext) {
  private[sdk] val _inputs: ArrayBuffer[ExtendedInputBox] = ArrayBuffer.empty
  private[sdk] val _outputs: ArrayBuffer[OutBox] = ArrayBuffer.empty
  private[sdk] val _dataInputs: ArrayBuffer[ErgoBox] = ArrayBuffer.empty

  private var _tokensToBurn: Option[ArrayBuffer[ErgoToken]] = None
  private var _feeAmount: Option[Long] = None
  private var _changeAddress: Option[ErgoAddress] = None
  private var _ph: Option[PreHeader] = None
}
```

### Builder Methods

Each method returns `this` for chaining:

```scala
// From UnsignedTransactionBuilder.scala:36-69
def addInputs(boxes: ExtendedInputBox*): this.type = {
  _inputs ++= boxes
  this
}

def addDataInputs(boxes: ErgoBox*): this.type = {
  _dataInputs ++= boxes
  this
}

def addOutputs(outBoxes: OutBox*): this.type = {
  _outputs ++= outBoxes
  this
}

def fee(feeAmount: Long): this.type = {
  require(_feeAmount.isEmpty, "Fee already defined")
  _feeAmount = Some(feeAmount)
  this
}

def addTokensToBurn(tokens: ErgoToken*): this.type = {
  if (_tokensToBurn.isEmpty)
    _tokensToBurn = Some(ArrayBuffer.empty)
  _tokensToBurn.get ++= tokens
  this
}

def sendChangeTo(changeAddress: ErgoAddress): this.type = {
  require(_changeAddress.isEmpty, "Change address is already specified")
  _changeAddress = Some(changeAddress)
  this
}
```

### Building the Transaction

```scala
// From UnsignedTransactionBuilder.scala:79-111
def build(): UnreducedTransaction = {
  val boxesToSpend = _inputs.toIndexedSeq
  val outputCandidates = _outputs.map(c => c.candidate).toIndexedSeq
  require(!outputCandidates.isEmpty, "Output boxes are not specified")

  val dataInputBoxes = _dataInputs.toIndexedSeq
  val dataInputs = _dataInputs.map(box => DataInput(box.id)).toIndexedSeq
  require(_feeAmount.isEmpty || _feeAmount.get >= BlockchainParameters.MinFee)

  val changeAddress = getDefined(_changeAddress, "Change address is not defined")
  val requestedToBurn = _tokensToBurn.fold(IndexedSeq.empty[ErgoToken])(_.toIndexedSeq)
  val burnTokens = SdkIsos.isoErgoTokenSeqToLinkedMap.to(requestedToBurn).toMap

  val rewardDelay = ctx.networkType match {
    case NetworkType.Mainnet => BlockchainParameters.MinerRewardDelay_Mainnet
    case NetworkType.Testnet => BlockchainParameters.MinerRewardDelay_Testnet
  }

  val tx = UnsignedTransactionBuilder.buildUnsignedTx(
    inputs = inputBoxesSeq, dataInputs = dataInputs,
    outputCandidates = outputCandidates,
    currentHeight = ctx.height, createFeeOutput = _feeAmount,
    changeAddress = changeAddress, minChangeValue = MinChangeValue,
    minerRewardDelay = rewardDelay, burnTokens = burnTokens
  ).get

  UnreducedTransaction(txWithExtensions, boxesToSpend, dataInputBoxes, requestedToBurn)
}
```

### Stateless Validation

Before building, stateless checks are performed:

```scala
// From UnsignedTransactionBuilder.scala:127-141
private def validateStatelessChecks(
    inputs: IndexedSeq[ErgoBox], dataInputs: IndexedSeq[DataInput],
    outputCandidates: Seq[ErgoBoxCandidate]): Unit = {
  require(inputs.nonEmpty, "inputs cannot be empty")
  require(outputCandidates.nonEmpty, "outputCandidates cannot be empty")
  require(inputs.size <= Short.MaxValue)
  require(dataInputs.size <= Short.MaxValue)
  require(outputCandidates.size <= Short.MaxValue)
  require(outputCandidates.forall(_.value >= 0))
  val outputSumTry = Try(outputCandidates.map(_.value).reduce(Math.addExact(_, _)))
  require(outputSumTry.isSuccess, "Sum should not exceed Long.MaxValue")
  require(inputs.distinct.size == inputs.size, "No duplicate inputs")
}
```

## OutBox Builder

For constructing output boxes:

```scala
// From OutBoxBuilder.scala:13-57
class OutBoxBuilder(val _txB: UnsignedTransactionBuilder) {
  private val _ctx = _txB.ctx
  private var _value: Long = 0
  private var _contract: ErgoTree = _
  private val _tokens = ArrayBuffer.empty[ErgoToken]
  private val _registers = ArrayBuffer.empty[Constant[_]]
  private var _creationHeightOpt: Option[Int] = None

  def value(value: Long): this.type = {
    _value = value
    this
  }

  def contract(contract: ErgoTree): this.type = {
    _contract = contract
    this
  }

  def tokens(tokens: ErgoToken*): this.type = {
    require(tokens.nonEmpty, "At least one token should be specified")
    val maxTokens = SigmaConstants.MaxTokens.value  // 255
    require(tokens.size <= maxTokens)
    _tokens ++= tokens
    this
  }

  def registers(registers: Constant[_]*): this.type = {
    require(registers.nonEmpty)
    _registers.clear()
    _registers ++= registers
    this
  }

  def build(): OutBox = {
    require(_contract != null, "Contract is not defined")
    val ergoBoxCandidate = OutBoxBuilder.createBoxCandidate(
      _value, _contract, _tokens.toSeq, _registers.toSeq,
      creationHeight = _creationHeightOpt.getOrElse(_txB.ctx.height))
    OutBox(ergoBoxCandidate)
  }
}
```

## Extended Input Box

Input boxes with context extensions:

```scala
// From ExtendedInputBox.scala:15-21
case class ExtendedInputBox(
  box: ErgoBox,
  extension: ContextExtension
) {
  def toUnsignedInput: UnsignedInput = new UnsignedInput(box.id, extension)
  def value: Long = box.value
}
```

## Transaction Data Types

### UnreducedTransaction

Before reduction (needs context for evaluation):

```scala
// From Transactions.scala:17-46
case class UnreducedTransaction(
    unsignedTx: UnsignedErgoLikeTransaction,
    boxesToSpend: IndexedSeq[ExtendedInputBox],
    dataInputs: IndexedSeq[ErgoBox],
    tokensToBurn: IndexedSeq[ErgoToken]
) {
  require(unsignedTx.inputs.length == boxesToSpend.length)
  require(unsignedTx.dataInputs.length == dataInputs.length)
  // Validates box IDs match between tx and actual boxes
}
```

### ReducedTransaction

After reduction (can be signed without context):

```scala
// From Transactions.scala:48-65
case class ReducedTransaction(ergoTx: ReducedErgoLikeTransaction) {
  def toHex: String = {
    val w = SigmaSerializer.startWriter()
    ReducedErgoLikeTransactionSerializer.serialize(ergoTx, w)
    w.toBytes.toHex
  }
}

object ReducedTransaction {
  def fromHex(hex: String): ReducedTransaction = {
    val r = SigmaSerializer.startReader(hex.toBytes)
    val tx = ReducedErgoLikeTransactionSerializer.parse(r)
    ReducedTransaction(tx)
  }
}
```

### SignedTransaction

Final signed transaction:

```scala
// From Transactions.scala:68
case class SignedTransaction(ergoTx: ErgoLikeTransaction, cost: Int)
```

## Reducing Interpreter

The `ReducingInterpreter` reduces scripts without signing:

```scala
// From ReducingInterpreter.scala:22-44
class ReducingInterpreter(params: BlockchainParameters) extends ErgoLikeInterpreter {
  override type CTX = ErgoLikeContext

  def reduce(env: ScriptEnv, ergoTree: ErgoTree, context: CTX): ReducedInputData = {
    val initCost = context.initCost
    val remainingLimit = context.costLimit - initCost
    if (remainingLimit <= 0)
      throw new CostLimitException(
        initCost,
        s"Estimated execution cost $initCost exceeds the limit ${context.costLimit}"
      )
    val ctxUpdInitCost = context.withInitCost(initCost)
    val res = fullReduction(ergoTree, ctxUpdInitCost, env)
    ReducedInputData(res, ctxUpdInitCost.extension)
  }
}
```

### Transaction Reduction

```scala
// From ReducingInterpreter.scala:58-158
def reduceTransaction(
    unreducedTx: UnreducedTransaction,
    stateContext: BlockchainStateContext,
    baseCost: Int
): ReducedTransaction = {
  val unsignedTx = unreducedTx.unsignedTx
  val boxesToSpend = unreducedTx.boxesToSpend
  val dataBoxes = unreducedTx.dataInputs

  // Validate token balances
  val tokensToBurn = unreducedTx.tokensToBurn
  val inputTokens = boxesToSpend.flatMap(_.box.additionalTokens.toArray)
  val outputTokens = unsignedTx.outputCandidates.flatMap(_.additionalTokens.toArray)
  val tokenDiff = JavaHelpers.subtractTokens(outputTokens, inputTokens)

  if (tokenDiff.nonEmpty) {
    val (toBurn, toMint) = tokenDiff.partition(_._2 < 0)
    // Validate burning matches requested
    // Validate only one token minted
  }

  // Calculate initial cost
  val initialCost = ArithUtils.addExact(
    Interpreter.interpreterInitCost,
    boxesToSpend.size * params.inputCost,
    dataBoxes.size * params.dataInputCost,
    unsignedTx.outputCandidates.size * params.outputCost
  )

  val maxCost = params.maxBlockCost
  var currentCost = addCostChecked(baseCost, initialCost, maxCost)

  // Add token access costs
  val (outAssets, outAssetsNum) = JavaHelpers.extractAssets(outputCandidates)
  val (inAssets, inAssetsNum) = JavaHelpers.extractAssets(boxesToSpend.map(_.box))
  val totalAssetsAccessCost = (outAssetsNum + inAssetsNum + inAssets.size + outAssets.size) *
                              params.tokenAccessCost
  currentCost = addCostChecked(currentCost, totalAssetsAccessCost, maxCost)

  // Reduce each input
  val reducedInputs = mutable.ArrayBuilder.make[ReducedInputData]
  for ((inputBox, boxIdx) <- boxesToSpend.zipWithIndex) {
    val context = new ErgoLikeContext(
      AvlTreeData.avlTreeFromDigest(stateContext.previousStateDigest),
      stateContext.sigmaLastHeaders,
      stateContext.sigmaPreHeader,
      transactionContext.dataBoxes,
      transactionContext.boxesToSpend,
      transactionContext.spendingTransaction,
      boxIdx.toShort,
      inputBox.extension,
      ValidationRules.currentSettings,
      costLimit = maxCost,
      initCost = currentCost,
      activatedScriptVersion = (params.blockVersion - 1).toByte
    )

    val reducedInput = reduce(Interpreter.emptyEnv, inputBox.box.ergoTree, context)
    currentCost = reducedInput.reductionResult.cost
    reducedInputs += reducedInput
  }

  ReducedTransaction(ReducedErgoLikeTransaction(
    unsignedTx, reducedInputs.result(),
    cost = (currentCost - baseCost).toIntExact
  ))
}
```

### ReducedInputData

The result of script reduction:

```scala
// From AppkitProvingInterpreter.scala:274-275
case class ReducedInputData(
  reductionResult: ReductionResult,
  extension: ContextExtension
)
```

### ReducedErgoLikeTransaction

Reduced transaction with sigma propositions:

```scala
// From AppkitProvingInterpreter.scala:283-289
case class ReducedErgoLikeTransaction(
  unsignedTx: UnsignedErgoLikeTransaction,
  reducedInputs: Seq[ReducedInputData],
  cost: Int
) {
  require(unsignedTx.inputs.length == reducedInputs.length)
}
```

## SigmaProver

High-level prover interface:

```scala
// From SigmaProver.scala (conceptual structure)
class SigmaProver(_prover: AppkitProvingInterpreter, networkPrefix: NetworkPrefix) {

  /** Get P2PK address for the first public key */
  def getP2PKAddress: P2PKAddress = {
    val pk = _prover.pubKeys(0)
    P2PKAddress(pk)
  }

  /** Reduce and sign a transaction */
  def sign(
      stateCtx: BlockchainStateContext,
      tx: UnreducedTransaction,
      baseCost: Int
  ): SignedTransaction

  /** Sign an arbitrary message with a sigma proposition */
  def signMessage(
      sigmaProp: SigmaProp,
      message: Array[Byte],
      hintsBag: HintsBag
  ): Array[Byte]

  /** Reduce transaction without signing (for cold wallet) */
  def reduce(
      stateCtx: BlockchainStateContext,
      tx: UnreducedTransaction,
      baseCost: Int
  ): ReducedTransaction

  /** Sign an already-reduced transaction */
  def signReduced(tx: ReducedTransaction): SignedTransaction
}
```

## ProverBuilder

Builder for creating SigmaProver instances:

```scala
// From ProverBuilder.scala:14-55
class ProverBuilder(parameters: BlockchainParameters, networkPrefix: NetworkPrefix) {
  private var _masterKey: Option[ExtendedSecretKey] = None
  private val _eip3Secrets = mutable.ArrayBuilder.make[ExtendedSecretKey]
  private val _dhtInputs = mutable.ArrayBuilder.make[DiffieHellmanTupleProverInput]
  private val _dLogInputs = mutable.ArrayBuilder.make[DLogProverInput]

  /** Configure with BIP-39 mnemonic phrase */
  def withMnemonic(
      mnemonicPhrase: SecretString,
      mnemonicPass: SecretString,
      usePre1627KeyDerivation: Boolean
  ): ProverBuilder = {
    _masterKey = Some(JavaHelpers.seedToMasterKey(
      mnemonicPhrase, mnemonicPass, usePre1627KeyDerivation))
    this
  }

  /** Add EIP-3 derived secret at given index */
  def withEip3Secret(index: Int): ProverBuilder = {
    require(_masterKey.isDefined, "Mnemonic is not defined")
    val path = JavaHelpers.eip3DerivationWithLastIndex(index)
    val secretKey = _masterKey.get.derive(path)
    _eip3Secrets += secretKey
    this
  }

  /** Add Diffie-Hellman tuple secret */
  def withDHTData(
      g: GroupElement, h: GroupElement,
      u: GroupElement, v: GroupElement,
      x: BigInteger
  ): ProverBuilder = {
    val dht = DiffieHellmanTupleProverInput(x, ProveDHTuple(g, h, u, v))
    _dhtInputs += dht
    this
  }

  /** Add discrete log secret */
  def withDLogSecret(x: BigInteger): ProverBuilder = {
    val dlog = DLogProverInput(x)
    _dLogInputs += dlog
    this
  }

  /** Build the prover */
  def build(): SigmaProver = {
    val interpreter = new AppkitProvingInterpreter(
      _eip3Secrets.result(), _dLogInputs.result(),
      _dhtInputs.result(), parameters)
    new SigmaProver(interpreter, networkPrefix)
  }
}
```

## AppkitProvingInterpreter

The core proving implementation:

```scala
// From AppkitProvingInterpreter.scala:34-55
class AppkitProvingInterpreter(
    val secretKeys: IndexedSeq[ExtendedSecretKey],
    val dLogInputs: IndexedSeq[DLogProverInput],
    val dhtInputs: IndexedSeq[DiffieHellmanTupleProverInput],
    params: BlockchainParameters)
  extends ReducingInterpreter(params) with ProverInterpreter {

  override type CTX = ErgoLikeContext

  /** All available secrets */
  override val secrets: Seq[SigmaProtocolPrivateInput[_]] = {
    val dlogs: IndexedSeq[DLogProverInput] = secretKeys.map(_.privateInput)
    dlogs ++ dLogInputs ++ dhtInputs
  }

  /** Public keys for dlog secrets */
  val pubKeys: Seq[ProveDlog] = secrets
      .filter { case _: DLogProverInput => true case _ => false }
      .map(_.asInstanceOf[DLogProverInput].publicImage)
}
```

### Sign Method

```scala
// From AppkitProvingInterpreter.scala:81-95
def sign(unreducedTx: UnreducedTransaction,
         stateContext: BlockchainStateContext,
         baseCost: Int): Try[SignedTransaction] = Try {
  val maxCost = params.maxBlockCost
  var currentCost: Long = baseCost

  // First reduce
  val reducedTx = reduceTransaction(unreducedTx, stateContext, baseCost)
  currentCost = addCostLimited(currentCost, reducedTx.ergoTx.cost, maxCost)

  // Then sign
  val signedTx = signReduced(reducedTx, currentCost.toInt)
  currentCost += signedTx.cost

  val reductionAndVerificationCost = (currentCost - baseCost).toIntExact
  signedTx.copy(cost = reductionAndVerificationCost)
}
```

### Sign Reduced

```scala
// From AppkitProvingInterpreter.scala:220-244
def signReduced(reducedTx: ReducedTransaction, baseCost: Int): SignedTransaction = {
  val provedInputs = mutable.ArrayBuilder.make[Input]
  val unsignedTx = reducedTx.ergoTx.unsignedTx

  val maxCost = params.maxBlockCost
  var currentCost: Long = baseCost

  for ((reducedInput, boxIdx) <- reducedTx.ergoTx.reducedInputs.zipWithIndex) {
    val unsignedInput = unsignedTx.inputs(boxIdx)

    // Generate proof for this input
    val proverResult = proveReduced(reducedInput, unsignedTx.messageToSign)
    val signedInput = Input(unsignedInput.boxId, proverResult)

    // Estimate verification cost
    val verificationCost = estimateCryptoVerifyCost(
      reducedInput.reductionResult.value).toBlockCost
    currentCost = addCostLimited(currentCost, verificationCost, maxCost)

    provedInputs += signedInput
  }

  val signedTx = new ErgoLikeTransaction(
    provedInputs.result(), unsignedTx.dataInputs, unsignedTx.outputCandidates)

  val txVerificationCost = (currentCost - baseCost).toIntExact
  SignedTransaction(signedTx, txVerificationCost)
}
```

### Prove Reduced

```scala
// From AppkitProvingInterpreter.scala:251-257
def proveReduced(
      reducedInput: ReducedInputData,
      message: Array[Byte],
      hintsBag: HintsBag = HintsBag.empty): ProverResult = {
  val proof = generateProof(reducedInput.reductionResult.value, message, hintsBag)
  new ProverResult(proof, reducedInput.extension)
}
```

## JSON Codecs

The SDK includes comprehensive JSON serialization:

### Basic Type Codecs

```scala
// From JsonCodecs.scala:67-84
implicit val arrayBytesEncoder: Encoder[Array[Byte]] =
  Encoder.instance(ErgoAlgos.encode(_).asJson)
implicit val arrayBytesDecoder: Decoder[Array[Byte]] =
  bytesDecoder(x => x)

implicit val collBytesEncoder: Encoder[Coll[Byte]] =
  Encoder.instance(ErgoAlgos.encode(_).asJson)
implicit val collBytesDecoder: Decoder[Coll[Byte]] =
  bytesDecoder(Colls.fromArray(_))

implicit val sigmaBigIntEncoder: Encoder[sigma.BigInt] = Encoder.instance({ bigInt =>
  JsonNumber.fromDecimalStringUnsafe(
    bigInt.asInstanceOf[WrapperOf[BigInteger]].wrappedValue.toString
  ).asJson
})
```

### ErgoTree Codec

```scala
// From JsonCodecs.scala:284-296
implicit val ergoTreeEncoder: Encoder[ErgoTree] = Encoder.instance({ value =>
  ErgoTreeSerializer.DefaultSerializer.serializeErgoTree(value).asJson
})

def decodeErgoTree[T](transform: ErgoTree => T): Decoder[T] =
  Decoder.instance({ implicit cursor: ACursor =>
    cursor.as[Array[Byte]] flatMap { bytes =>
      fromThrows(transform(
        ErgoTreeSerializer.DefaultSerializer.deserializeErgoTree(bytes)))
    }
  })
```

### ErgoBox Codec

```scala
// From JsonCodecs.scala:309-340
implicit val ergoBoxEncoder: Encoder[ErgoBox] = Encoder.instance({ box =>
  Json.obj(
    "boxId" -> box.id.asJson,
    "value" -> box.value.asJson,
    "ergoTree" -> ErgoTreeSerializer.DefaultSerializer
                    .serializeErgoTree(box.ergoTree).asJson,
    "assets" -> box.additionalTokens.toArray.toSeq.asJson,
    "creationHeight" -> box.creationHeight.asJson,
    "additionalRegisters" -> box.additionalRegisters.asJson,
    "transactionId" -> box.transactionId.asJson,
    "index" -> box.index.asJson
  )
})

implicit val ergoBoxDecoder: Decoder[ErgoBox] = Decoder.instance({ cursor =>
  for {
    value <- cursor.downField("value").as[Long]
    ergoTreeBytes <- cursor.downField("ergoTree").as[Array[Byte]]
    additionalTokens <- cursor.downField("assets").as(Decoder.decodeSeq(assetDecoder))
    creationHeight <- cursor.downField("creationHeight").as[Int]
    additionalRegisters <- cursor.downField("additionalRegisters").as(registersDecoder)
    transactionId <- cursor.downField("transactionId").as[ModifierId]
    index <- cursor.downField("index").as[Short]
  } yield new ErgoBox(value = value, ergoTree = ..., ...)
})
```

### Transaction Codecs

```scala
// From JsonCodecs.scala:368-383
implicit val ergoLikeTransactionEncoder: Encoder[ErgoLikeTransaction] =
  Encoder.instance({ tx =>
    Json.obj(
      "id" -> tx.id.asJson,
      "inputs" -> tx.inputs.asJson,
      "dataInputs" -> tx.dataInputs.asJson,
      "outputs" -> tx.outputs.asJson
    )
  })

implicit val ergoLikeTransactionDecoder: Decoder[ErgoLikeTransaction] =
  Decoder.instance({ implicit cursor =>
    for {
      inputs <- cursor.downField("inputs").as[IndexedSeq[Input]]
      dataInputs <- cursor.downField("dataInputs").as[IndexedSeq[DataInput]]
      outputs <- cursor.downField("outputs").as[IndexedSeq[ErgoBoxCandidate]]
    } yield new ErgoLikeTransaction(inputs, dataInputs, outputs)
  })
```

### Context Codec

```scala
// From JsonCodecs.scala:423-459
implicit val ergoLikeContextEncoder: Encoder[ErgoLikeContext] =
  Encoder.instance({ ctx =>
    Json.obj(
      "lastBlockUtxoRoot" -> ctx.lastBlockUtxoRoot.asJson,
      "headers" -> ctx.headers.toArray.toSeq.asJson,
      "preHeader" -> ctx.preHeader.asJson,
      "dataBoxes" -> ctx.dataBoxes.asJson,
      "boxesToSpend" -> ctx.boxesToSpend.asJson,
      "spendingTransaction" -> ctx.spendingTransaction.asJson,
      "selfIndex" -> ctx.selfIndex.asJson,
      "extension" -> ctx.extension.asJson,
      "validationSettings" -> ctx.validationSettings.asJson,
      "costLimit" -> ctx.costLimit.asJson,
      "initCost" -> ctx.initCost.asJson,
      "scriptVersion" -> ctx.activatedScriptVersion.asJson
    )
  })
```

## Box Selection

The SDK includes box selection for transaction building:

```scala
// From BoxSelectionResult.scala:12-15
class BoxSelectionResult[T <: ErgoBoxAssets](
    val inputBoxes: Seq[T],
    val changeBoxes: Seq[ErgoBoxAssets],
    val payToReemissionBox: Option[ErgoBoxAssets]  // EIP-27
)
```

## Token Balance Exception

For token mismatch errors:

```scala
// From AppkitProvingInterpreter.scala:264-267
case class TokenBalanceException(
  message: String,
  tokensDiff: TokenColl
) extends Exception(s"Input and output tokens are not balanced: $message")
```

## Usage Example

Complete transaction building and signing:

```scala
import org.ergoplatform.sdk._

// Create blockchain context
val ctx = BlockchainContext(
  NetworkType.Mainnet,
  parameters,
  stateContext
)

// Build prover
val prover = new ProverBuilder(parameters, NetworkPrefix.Mainnet)
  .withMnemonic(mnemonic, password, usePre1627KeyDerivation = false)
  .withEip3Secret(0)
  .build()

// Build transaction
val tx = new UnsignedTransactionBuilder(ctx)
  .addInputs(inputBox1, inputBox2)
  .addOutputs(
    ctx.outBoxBuilder
      .value(1000000000L)  // 1 ERG
      .contract(recipientTree)
      .build()
  )
  .fee(BlockchainParameters.MinFee)
  .sendChangeTo(changeAddress)
  .build()

// Sign transaction
val signedTx = prover.sign(ctx.stateContext, tx, baseCost = 0)
println(s"Signed TX cost: ${signedTx.cost}")
```

### Cold Wallet Flow

Using reduction for air-gapped signing:

```scala
// On hot wallet: reduce transaction
val reducedTx = prover.reduce(stateContext, unreducedTx, baseCost = 0)
val reducedHex = reducedTx.toHex

// Transfer hex string to cold wallet (QR code, USB, etc.)

// On cold wallet: sign reduced transaction
val reducedTx = ReducedTransaction.fromHex(reducedHex)
val signedTx = coldProver.signReduced(reducedTx)
```

## Serialization of Reduced Transactions

For cold wallet workflows:

```scala
// From AppkitProvingInterpreter.scala:292-336
object ReducedErgoLikeTransactionSerializer
  extends SigmaSerializer[ReducedErgoLikeTransaction, ReducedErgoLikeTransaction] {

  override def serialize(tx: ReducedErgoLikeTransaction, w: SigmaByteWriter): Unit = {
    val msg = tx.unsignedTx.messageToSign
    w.putUInt(msg.length)
    w.putBytes(msg)

    val nInputs = tx.reducedInputs.length
    cfor(0)(_ < nInputs, _ + 1) { i =>
      val input = tx.reducedInputs(i)
      SigmaBoolean.serializer.serialize(input.reductionResult.value, w)
      w.putULong(input.reductionResult.cost)
    }
    w.putUInt(tx.cost)
  }

  override def parse(r: SigmaByteReader): ReducedErgoLikeTransaction = {
    val nBytes = r.getUInt()
    val msg = r.getBytes(nBytes.toIntExact)
    val tx = ErgoLikeTransactionSerializer.parse(SigmaSerializer.startReader(msg))

    val nInputs = tx.inputs.length
    val reducedInputs = new Array[ReducedInputData](nInputs)
    val unsignedInputs = new Array[UnsignedInput](nInputs)

    cfor(0)(_ < nInputs, _ + 1) { i =>
      val sb = SigmaBoolean.serializer.parse(r)
      val cost = r.getULong()
      val input = tx.inputs(i)
      val extension = input.extension
      val reductionResult = ReductionResult(sb, cost)
      reducedInputs(i) = ReducedInputData(reductionResult, extension)
      unsignedInputs(i) = new UnsignedInput(input.boxId, extension)
    }

    val cost = r.getUIntExact
    val unsignedTx = UnsignedErgoLikeTransaction(
      unsignedInputs, tx.dataInputs, tx.outputCandidates)
    ReducedErgoLikeTransaction(unsignedTx, reducedInputs, cost)
  }
}
```

## Exercises

1. **Implementation**: Build a transaction that sends tokens to multiple recipients using `UnsignedTransactionBuilder`.

2. **Analysis**: Trace through the cost calculation in `reduceTransaction`. What are all the components of the total cost?

3. **Design**: How would you implement a multi-signature signing flow using reduced transactions?

4. **Extension**: Implement a method to estimate the minimum fee required for a transaction based on its structure.

## Summary

- **UnsignedTransactionBuilder** provides fluent API for constructing transactions
- **OutBoxBuilder** creates output boxes with tokens and registers
- **ReducingInterpreter** reduces scripts to sigma propositions without secrets
- **SigmaProver** combines reduction and signing into a simple interface
- **JSON codecs** enable API integration with external systems
- **Reduced transactions** enable cold wallet signing workflows

## Further Reading

- Source: `sdk/shared/src/main/scala/org/ergoplatform/sdk/`
- EIP-3 (Address Generation): https://github.com/ergoplatform/eips/blob/master/eip-0003.md
- Ergo Appkit Documentation: https://github.com/ergoplatform/ergo-appkit

---
*[Previous: Chapter 26](../part8/ch26-wallet-signing.md) | [Next: Chapter 28](./ch28-key-derivation.md)*
