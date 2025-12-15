# The Sigma Book

## Complete Technical Reference for SigmaState Interpreter

Welcome to **The Sigma Book**, a comprehensive technical reference covering the SigmaState interpreter, ErgoTrees, and the Sigma language. This book is written for engineers who need deep understanding of the implementation details, algorithms, and design decisions behind the Ergo blockchain's smart contract system.

## What This Book Covers

This book provides complete documentation of:

1. **Specifications**: The formal and informal specifications of the Sigma language, type system, and ErgoTree format
2. **Implementation Details**: Internal algorithms, data structures, and engineering decisions in the reference Scala implementation
3. **Node Integration**: How the Ergo node uses the interpreter for transaction validation
4. **Practical APIs**: The SDK and high-level interfaces for building applications

## How to Read This Book

### Prerequisites Approach

Every chapter includes an explicit **Prerequisites** section that lists:
- Required knowledge assumptions
- Related concepts you should understand
- Links to earlier chapters covering dependencies

This allows you to:
- Jump directly to topics of interest if you have the background
- Trace backward to fill gaps in your understanding
- Use the book as a reference rather than reading linearly

### Code Examples

All code examples in this book are designed to be runnable. They use the Scala reference implementation and can be executed with the sigmastate-interpreter library.

### Exercises

Each chapter concludes with exercises at three levels:
1. **Conceptual**: Test your understanding of the material
2. **Implementation**: Write code applying the concepts
3. **Analysis**: Read and analyze real source code

## Source Material

This book is derived directly from:

- **sigmastate-interpreter**: The reference Scala implementation
- **Ergo node**: The full node implementation showing integration
- **Formal specifications**: LaTeX documents and academic papers
- **Test suites**: Comprehensive tests that define expected behavior

All claims in this book are verified against these primary sources.

## Book Structure

| Part | Focus | Depth |
|------|-------|-------|
| I. Foundations | Core concepts and type system | Overview |
| II. AST | Expression node catalog | Reference |
| III. Serialization | Binary format | Detailed |
| IV. Cryptography | Zero-knowledge proofs | **Deep** |
| V. Interpreter | Evaluation engine | **Deep** |
| VI. Compiler | ErgoScript compilation | **Deep** |
| VII. Data Structures | Collections, AVL trees, boxes | Detailed |
| VIII. Node Integration | Transaction validation | Practical |
| IX. SDK | Developer APIs | Practical |
| X. Advanced | Soft-forks, cross-platform | Specialized |

## Conventions Used

```scala
// Code blocks show Scala from the reference implementation
val ergoTree: ErgoTree = ???
```

> **Note**: Highlighted notes provide important context or warnings.

**Source reference**: `path/to/file.scala:123` - Links to specific source locations.

## Version Information

This book documents:
- **sigmastate-interpreter**: Version 6.x (with notes on v5 differences)
- **Ergo node**: Compatible versions
- **Protocol versions**: v0, v1, v2 (AOT), v3 (JIT)

## Contributing

This book is maintained as part of the ErgoTree research project. Corrections and improvements are welcome.

---

*Let's begin with [Chapter 1: Introduction to Sigma and ErgoTree](./part1/ch01-introduction.md).*
