# Glossary

## A

**AOT (Ahead-Of-Time)**: Costing model where script costs are calculated before execution. Used in ErgoTree versions 0-1.

**AVL Tree**: A self-balancing binary search tree used for authenticated dictionaries in Ergo.

## B

**BigInt**: 256-bit signed integer type in ErgoTree.

**Box**: The fundamental UTXO unit in Ergo, containing value, ErgoTree script, tokens, and registers.

## C

**Constant Segregation**: Optimization where constants are extracted from ErgoTree expressions and stored in a separate array. Enables efficient script substitution without re-serializing the expression tree.

**Context**: Execution environment containing blockchain state (HEIGHT, headers), transaction data (INPUTS, OUTPUTS, dataInputs), and current input information (SELF).

**Cost Accumulator**: Runtime tracker that sums operation costs and enforces the script cost limit.

## D

**Data Input**: Read-only box reference in a transaction. Provides data without being spent.

**DHT (Diffie-Hellman Tuple)**: Four-element sigma protocol proving knowledge of secret x where u = g^x and v = h^x.

**DLog (Discrete Logarithm)**: Sigma protocol proving knowledge of discrete logarithm. Given generator g and public key h = g^x, proves knowledge of x.

## E

**ErgoScript**: High-level smart contract language with Scala-like syntax.

**ErgoTree**: Serialized bytecode representation of smart contracts.

## F

**Fiat-Shamir Transformation**: Technique to convert interactive proofs into non-interactive proofs.

## G

**GroupElement**: An elliptic curve point on secp256k1.

## H

**Header**: The first byte(s) of ErgoTree that specify version and format flags.

## I

**Interpreter**: Component that evaluates ErgoTree expressions against a context to produce a SigmaBoolean result.

## J

**JIT (Just-In-Time)**: Costing model where costs are calculated during execution. Used in ErgoTree version 2+.

## O

**OpCode**: Single-byte identifier for expression nodes in serialized ErgoTree. Values 0x01-0x70 encode constants; 0x71+ encode operations.

## P

**Prover**: Component that generates cryptographic proofs for spending conditions.

**Proposition**: A statement that can be proven true or false.

## S

**Secp256k1**: The elliptic curve used in Ergo (same as Bitcoin).

**SigmaBoolean**: A tree of cryptographic propositions (AND, OR, threshold, DLog, DHT).

**SigmaProp**: Type representing sigma-protocol propositions.

**Sigma Protocol**: Zero-knowledge proof system with three-move structure.

## T

**Type Code**: Unique byte identifier for each type in ErgoTree serialization.

## U

**UTXO**: Unspent Transaction Output model used by Ergo.

**UnsignedBigInt**: 256-bit unsigned integer type (added in v6).

## V

**Verifier**: Component that verifies cryptographic proofs.

**VLQ**: Variable-Length Quantity encoding for unsigned integers. Uses 7 data bits per byte with continuation bit.

## Z

**ZigZag Encoding**: Maps signed integers to unsigned: 0→0, -1→1, 1→2, -2→3, etc. Keeps small negatives small for efficient VLQ encoding.

---
*[Back to Contents](./SUMMARY.md)*
