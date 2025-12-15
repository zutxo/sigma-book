# Chapter 26: Wallet and Signing

## Prerequisites

- **Required knowledge**: Prover implementation (Chapter 15), Interpreter wrappers (Chapter 23)
- **Related concepts**: Sigma protocols (Chapter 11), Transaction validation (Chapter 24)
- **Prior chapters**: Chapter 25 (Cost Limits and Parameters)

## Learning Objectives

- Understand the wallet service architecture and operations
- Learn the transaction signing flow with `ErgoProvingInterpreter`
- Master distributed signing using `TransactionHintsBag`
- Understand asset extraction and cost calculation
- Learn wallet box management and selection

## Source References

- `ergo/src/main/scala/org/ergoplatform/nodeView/wallet/ErgoWalletService.scala`
- `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/TransactionHintsBag.scala`
- `ergo-wallet/src/main/scala/org/ergoplatform/wallet/boxes/ErgoBoxAssetExtractor.scala`
- `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/ErgoProvingInterpreter.scala`

## Introduction

The wallet layer bridges high-level user operations (send funds, sign transactions) with the low-level interpreter infrastructure. Key responsibilities:

1. **Key management**: Storing and deriving secret keys
2. **Box tracking**: Monitoring owned UTXOs
3. **Transaction building**: Assembling inputs/outputs
4. **Signing**: Generating proofs with `ErgoProvingInterpreter`
5. **Distributed signing**: Coordinating multi-party signatures via hints

## Wallet Service Interface

The `ErgoWalletService` trait defines the main wallet operations:

```scala
// From ErgoWalletService.scala:36-266
trait ErgoWalletService {
  val ergoSettings: ErgoSettings

  // Wallet lifecycle
  def readWallet(state: ErgoWalletState, ...): ErgoWalletState
  def initWallet(state: ErgoWalletState, ...): Try[(SecretString, ErgoWalletState)]
  def restoreWallet(state: ErgoWalletState, ...): Try[ErgoWalletState]
  def unlockWallet(state: ErgoWalletState, walletPass: SecretString, ...): Try[ErgoWalletState]
  def lockWallet(state: ErgoWalletState): ErgoWalletState

  // Box management
  def getWalletBoxes(state: ErgoWalletState, unspentOnly: Boolean, ...): Seq[WalletBox]
  def collectBoxes(state: ErgoWalletState, boxSelector: BoxSelector, ...): Try[CollectedBoxes]

  // Transaction operations
  def signTransaction(proverOpt: Option[ErgoProvingInterpreter], ...): Try[ErgoTransaction]
  def generateTransaction(state: ErgoWalletState, ...): Try[ErgoLikeTransactionTemplate[_]]

  // Distributed signing (EIP-11)
  def generateCommitments(state: ErgoWalletState, ...): Try[TransactionHintsBag]
  def extractHints(state: ErgoWalletState, ...): TransactionHintsBag
}
```

## Transaction Signing Flow

### Sign Transaction Method

```scala
// From ErgoWalletService.scala:226-233
def signTransaction(
    proverOpt: Option[ErgoProvingInterpreter],
    tx: UnsignedErgoTransaction,
    secrets: Seq[ExternalSecret],
    hints: TransactionHintsBag,
    boxesToSpendOpt: Option[Seq[ErgoBox]],
    dataBoxesOpt: Option[Seq[ErgoBox]],
    parameters: Parameters,
    stateContext: ErgoStateContext
)(extract: BoxId => Option[ErgoBox]): Try[ErgoTransaction]
```

### Signing Implementation

```scala
// From ErgoWalletService.scala:468-491
override def generateTransaction(
    state: ErgoWalletState,
    boxSelector: BoxSelector,
    requests: Seq[TransactionGenerationRequest],
    inputsRaw: Seq[String],
    dataInputsRaw: Seq[String],
    sign: Boolean
): Try[ErgoLikeTransactionTemplate[_]] = {

  val tx = generateUnsignedTransaction(state, boxSelector, requests, inputsRaw, dataInputsRaw)

  if (sign) {
    tx.flatMap { case (unsignedTx, inputs, dataInputs) =>
      state.walletVars.proverOpt match {
        case Some(prover) =>
          prover.sign(unsignedTx, inputs, dataInputs, state.stateContext, TransactionHintsBag.empty)
            .map(ErgoTransaction.apply)
            .fold(
              e => Failure(new Exception(s"Failed to sign: ${e.getMessage}")),
              tx => Success(tx)
            )
        case None =>
          Failure(new Exception("Wallet locked or not initialized"))
      }
    }
  } else {
    tx.map(_._1)  // Return unsigned transaction
  }
}
```

## TransactionHintsBag

The `TransactionHintsBag` manages hints for distributed signing (EIP-11):

```scala
// From TransactionHintsBag.scala:5-56
case class TransactionHintsBag(
    secretHints: Map[Int, HintsBag],  // Per-input secret hints
    publicHints: Map[Int, HintsBag]   // Per-input public hints
) {

  /** Replace hints for a specific input index */
  def replaceHintsForInput(index: Int, hintsBag: HintsBag): TransactionHintsBag = {
    val (secret, public) = hintsBag.hints.partition(_.isInstanceOf[OwnCommitment])
    TransactionHintsBag(
      secretHints.updated(index, HintsBag(secret)),
      publicHints.updated(index, HintsBag(public))
    )
  }

  /** Add hints for a specific input index (accumulate) */
  def addHintsForInput(index: Int, hintsBag: HintsBag): TransactionHintsBag = {
    val (secret, public) = hintsBag.hints.partition(_.isInstanceOf[OwnCommitment])
    val oldSecret = this.secretHints.getOrElse(index, HintsBag.empty)
    val oldPublic = this.publicHints.getOrElse(index, HintsBag.empty)

    TransactionHintsBag(
      secretHints.updated(index, HintsBag(secret) ++ oldSecret),
      publicHints.updated(index, HintsBag(public) ++ oldPublic)
    )
  }

  /** Get all hints (public + secret) for an input */
  def allHintsForInput(index: Int): HintsBag = {
    secretHints.getOrElse(index, HintsBag.empty) ++
    publicHints.getOrElse(index, HintsBag.empty)
  }
}

object TransactionHintsBag {
  val empty: TransactionHintsBag = new TransactionHintsBag(Map.empty, Map.empty)

  def apply(mixedHints: Map[Int, HintsBag]): TransactionHintsBag = {
    mixedHints.keys.foldLeft(TransactionHintsBag.empty) { case (thb, idx) =>
      thb.replaceHintsForInput(idx, mixedHints(idx))
    }
  }
}
```

### Hint Types

Hints are separated into:
- **Secret hints** (`OwnCommitment`): Prover's own commitments (randomness `r`)
- **Public hints**: Commitments and challenges from other signers

This separation enables secure distributed signing where secrets never leave their owner.

## Distributed Signing Protocol (EIP-11)

### Step 1: Generate Commitments

Each signer generates their commitments:

```scala
// From ErgoWalletService.scala:248-253
def generateCommitments(
    state: ErgoWalletState,
    unsignedTx: UnsignedErgoTransaction,
    externalSecretsOpt: Option[Seq[ExternalSecret]],
    externalInputsOpt: Option[Seq[ErgoBox]],
    externalDataInputsOpt: Option[Seq[ErgoBox]]
): Try[TransactionHintsBag]
```

Implementation:
```scala
// From ErgoWalletService.scala:493-499
override def generateCommitments(...): Try[TransactionHintsBag] = {
  val walletSecrets = state.walletVars.proverOpt.map(_.secretKeys).getOrElse(Seq.empty)
  val secrets = walletSecrets ++ externalSecretsOpt.getOrElse(Seq.empty).map(_.key)
  // ... create prover and generate commitments
}
```

### Step 2: Exchange Public Hints

Signers share their public commitments (without revealing secrets).

### Step 3: Sign with Combined Hints

Each signer signs using combined hints:

```scala
val combinedHints = myHints.addHintsForInput(0, otherSignerHints)
prover.sign(unsignedTx, inputs, dataInputs, stateContext, combinedHints)
```

### Step 4: Extract Hints from Partial Signatures

```scala
// From ErgoWalletService.scala:258-264
def extractHints(
    state: ErgoWalletState,
    tx: ErgoTransaction,
    real: Seq[SigmaBoolean],       // Propositions being proven
    simulated: Seq[SigmaBoolean],  // Propositions being simulated
    boxesToSpendOpt: Option[Seq[ErgoBox]],
    dataBoxesOpt: Option[Seq[ErgoBox]]
): TransactionHintsBag
```

## Asset Extraction

The `ErgoBoxAssetExtractor` handles token accounting:

```scala
// From ErgoBoxAssetExtractor.scala:11-41
object ErgoBoxAssetExtractor {
  val MaxAssetsPerBox = 255

  /** Extract total token amounts from boxes */
  def extractAssets(
      boxes: IndexedSeq[ErgoBoxCandidate]
  ): Try[(Map[Seq[Byte], Long], Int)] = Try {
    val map: mutable.Map[Seq[Byte], Long] = mutable.Map()
    val assetsNum = boxes.foldLeft(0) { case (acc, box) =>
      require(
        box.additionalTokens.length <= MaxAssetsPerBox,
        "too many assets in one box"
      )
      box.additionalTokens.foreach { case (assetId, amount) =>
        val total = map.getOrElse(assetId, 0L)
        map.put(assetId, Math.addExact(total, amount))
      }
      acc + box.additionalTokens.size
    }
    map.toMap -> assetsNum
  }
}
```

### Token Access Cost Calculation

```scala
// From ErgoBoxAssetExtractor.scala:55-65
def totalAssetsAccessCost(
    inAssetsNum: Int,     // Total input token slots
    inAssetsSize: Int,    // Unique input token IDs
    outAssetsNum: Int,    // Total output token slots
    outAssetsSize: Int,   // Unique output token IDs
    tokenAccessCost: Int  // Cost per access
): Int = {
  // Cost to iterate through all tokens
  val allAssetsCost = Math.multiplyExact(
    Math.addExact(outAssetsNum, inAssetsNum),
    tokenAccessCost
  )
  // Cost to check preservation of unique tokens
  val uniqueAssetsCost = Math.multiplyExact(
    Math.addExact(inAssetsSize, outAssetsSize),
    tokenAccessCost
  )
  Math.addExact(allAssetsCost, uniqueAssetsCost)
}
```

## Box Collection and Selection

### Collecting Boxes for Spending

```scala
// From ErgoWalletService.scala:451-466
override def collectBoxes(
    state: ErgoWalletState,
    boxSelector: BoxSelector,
    targetBalance: Long,
    targetAssets: Map[ErgoBox.TokenId, Long]
): Try[CollectedBoxes] = {
  val assetsMap = targetAssets.map(t => t._1.toModifierId -> t._2)
  val inputBoxes = state.getBoxesToSpend

  boxSelector
    .select(inputBoxes.iterator, state.walletFilter, targetBalance, assetsMap)
    .leftMap(m => new Exception(m.message))
    .map { res =>
      val ergoBoxes = res.inputBoxes.map(_.box)
      val changeBoxes = res.changeBoxes.map(b => ChangeBox(b.value, b.tokens))
      CollectedBoxes(ergoBoxes, changeBoxes)
    }.toTry
}
```

### Getting Wallet Boxes

```scala
// From ErgoWalletService.scala:399-418
override def getWalletBoxes(
    state: ErgoWalletState,
    unspentOnly: Boolean,
    considerUnconfirmed: Boolean
): Seq[WalletBox] = {
  val currentHeight = state.fullHeight
  val boxes = if (unspentOnly) {
    val confirmed = state.registry.walletUnspentBoxes(
      state.maxInputsToUse * BoxSelector.ScanDepthFactor
    )
    if (considerUnconfirmed) {
      // Filter out spent boxes, add off-chain boxes
      (confirmed ++ state.offChainRegistry.offChainBoxes).filter(state.walletFilter)
    } else {
      confirmed
    }
  } else {
    val confirmed = state.registry.walletConfirmedBoxes()
    if (considerUnconfirmed) {
      confirmed ++ state.offChainRegistry.offChainBoxes
    } else {
      confirmed
    }
  }
  boxes.map(tb => WalletBox(tb, currentHeight)).sortBy(_.trackedBox.inclusionHeightOpt)
}
```

## Wallet Lifecycle

### Initialization

```scala
// From ErgoWalletService.scala:300-328
override def initWallet(
    state: ErgoWalletState,
    settings: ErgoSettings,
    walletPass: SecretString,
    mnemonicPassOpt: Option[SecretString]
): Try[(SecretString, ErgoWalletState)] = {
  val walletSettings = settings.walletSettings
  // Generate high-quality random entropy
  val entropy = scorex.utils.Random.randomBytes(walletSettings.seedStrengthBits / 8)

  def initStorage(mnemonic: SecretString): Try[JsonSecretStorage] =
    Try(JsonSecretStorage.init(
      Mnemonic.toSeed(mnemonic, mnemonicPassOpt),
      walletPass,
      usePre1627KeyDerivation = false
    )(walletSettings.secretStorage))

  val result = new Mnemonic(...)
    .toMnemonic(entropy)
    .flatMap { mnemonic =>
      initStorage(mnemonic).flatMap { newSecretStorage =>
        // Clean up old wallet state
        recreateRegistry(state, settings).flatMap { stateV1 =>
          recreateStorage(stateV1, settings).map { stateV2 =>
            mnemonic -> stateV2.copy(secretStorageOpt = Some(newSecretStorage))
          }
        }
      }
    }

  // Securely erase entropy from memory
  java.util.Arrays.fill(entropy, 0: Byte)
  result
}
```

### Unlock and Lock

```scala
// From ErgoWalletService.scala:350-376
override def unlockWallet(
    state: ErgoWalletState,
    walletPass: SecretString,
    usePreEip3Derivation: Boolean
): Try[ErgoWalletState] = {
  if (state.walletVars.proverOpt.isEmpty) {
    state.secretStorageOpt match {
      case Some(secretStorage) =>
        secretStorage.unlock(walletPass).flatMap { _ =>
          secretStorage.secret match {
            case None =>
              Failure(new Exception("Master key not available"))
            case Some(masterKey) =>
              processUnlock(state, masterKey, usePreEip3Derivation)
          }
        }
      case None =>
        Failure(new Exception("Wallet not initialized"))
    }
  } else {
    Failure(new Exception("Wallet already unlocked"))
  }
}

override def lockWallet(state: ErgoWalletState): ErgoWalletState = {
  state.secretStorageOpt.foreach(_.lock())
  state.copy(walletVars = state.walletVars.resetProver())
}
```

## Complete Signing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                   Transaction Signing Flow                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. User Request                                            │
│     └─→ collectBoxes(targetBalance, targetAssets)          │
│         └─→ BoxSelector selects inputs + change            │
│                                                             │
│  2. Build Unsigned Transaction                              │
│     └─→ inputs, dataInputs, outputCandidates               │
│                                                             │
│  3. Sign Transaction                                        │
│     └─→ ErgoProvingInterpreter.sign()                      │
│         ├─→ Calculate initial cost                         │
│         ├─→ For each input:                                │
│         │   ├─→ Create ErgoLikeContext                     │
│         │   ├─→ Get hints for input                        │
│         │   ├─→ prove(ergoTree, context, message, hints)   │
│         │   └─→ Accumulate cost                            │
│         └─→ Return signed transaction                       │
│                                                             │
│  4. Broadcast                                               │
│     └─→ Submit to mempool                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Usage Example: Distributed Signing

```scala
// Party A: Generate commitments
val commitmentsA = walletA.generateCommitments(unsignedTx, secretsA, inputs, dataInputs)

// Party B: Generate commitments
val commitmentsB = walletB.generateCommitments(unsignedTx, secretsB, inputs, dataInputs)

// Exchange public hints
val publicHintsA = commitmentsA.publicHints
val publicHintsB = commitmentsB.publicHints

// Party A: Sign with combined hints
val combinedHintsA = commitmentsA.addHintsForInput(0, HintsBag(publicHintsB(0).hints))
val partialSigA = proverA.sign(unsignedTx, inputs, dataInputs, stateContext, combinedHintsA)

// Party B: Extract hints from A's partial signature
val extractedHints = walletB.extractHints(partialSigA, realProps, simulatedProps, inputs, dataInputs)

// Party B: Complete signing
val finalTx = proverB.sign(unsignedTx, inputs, dataInputs, stateContext,
  commitmentsB.addHintsForInput(0, extractedHints.allHintsForInput(0)))
```

## Exercises

1. **Conceptual**: Why are hints separated into "secret" and "public" categories in `TransactionHintsBag`?

2. **Implementation**: Write code to calculate the total asset access cost for a transaction with 3 inputs containing 5 token types total and 2 outputs containing 3 token types.

3. **Analysis**: In the distributed signing protocol, what information is safe to share and what must remain secret?

4. **Design**: How would you extend the wallet service to support hardware wallet integration where signing happens on a separate device?

## Summary

- **ErgoWalletService** manages the complete wallet lifecycle
- **TransactionHintsBag** enables distributed signing by separating public/secret hints
- **Asset extraction** calculates token costs for validation
- **Box selection** finds optimal input set for transactions
- **Signing flow** coordinates `ErgoProvingInterpreter` with context and hints

## Further Reading

- Source: `ergo/src/main/scala/org/ergoplatform/nodeView/wallet/ErgoWalletService.scala`
- Source: `ergo-wallet/src/main/scala/org/ergoplatform/wallet/interpreter/TransactionHintsBag.scala`
- EIP-11: Distributed Signing Protocol: https://github.com/ergoplatform/eips/blob/master/eip-0011.md

---
*[Previous: Chapter 25](./ch25-cost-limits-parameters.md) | [Next: Chapter 27](../part9/ch27-high-level-sdk.md)*
