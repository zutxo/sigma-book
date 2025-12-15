# The Sigma Book

> ## KYA: Know Your Assumptions
>
> **This is a PRE-ALPHA version of The Sigma Book.**
>
> Before using this material, understand these critical assumptions:
>
> 1. **Not Authoritative**: This book is NOT an official specification. It is a research and educational resource derived from studying the source code.
>
> 2. **May Contain Errors**: Content has not been formally verified. Implementations based solely on this book may be incorrect.
>
> 3. **Subject to Change**: As a pre-alpha work, chapters may be incomplete, reorganized, or substantially rewritten.
>
> 4. **Source of Truth**: For authoritative information, always consult:
>    - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
>    - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
>    - [ergo](https://github.com/ergoplatform/ergo) — Ergo node
>    - [ErgoTree Specification (spec.pdf)](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/docs/spec/spec.pdf)
>
> 5. **Verification Required**: Cross-reference all claims against the actual source code before relying on them.
>
> **Use this book to learn and explore, but verify everything against the source.**

---

## Complete Technical Reference for SigmaState Interpreter

Welcome to **The Sigma Book**, a comprehensive technical reference covering the SigmaState interpreter, ErgoTrees, and the Sigma language. This book is written for engineers who need deep understanding of the implementation details, algorithms, and data structures behind the Ergo blockchain's smart contract system.

Code examples use idiomatic **Zig 0.13+** with data-oriented design patterns, making algorithms explicit and accessible to implementers in any language.

## What This Book Covers

This book provides complete documentation of:

1. **Specifications**: Formal and informal specifications of the Sigma language, type system, and ErgoTree format
2. **Implementation Details**: Internal algorithms and data structures from both the reference Scala implementation (sigmastate-interpreter) and Rust implementation (sigma-rust)
3. **Node Integration**: How the Ergo node uses the interpreter for transaction validation
4. **Practical APIs**: SDK and high-level interfaces for building applications

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

Code examples use Zig 0.13+ to illustrate algorithms with explicit memory management and data-oriented patterns. While not directly runnable against the Scala or Rust implementations, they demonstrate the core logic clearly.

### Exercises

Each chapter concludes with exercises at three levels:
1. **Conceptual**: Test your understanding of the material
2. **Implementation**: Write code applying the concepts
3. **Analysis**: Read and analyze real source code

## Source Material

This book is derived from:

- **sigmastate-interpreter**: Reference Scala implementation (ScorexFoundation/sigmastate-interpreter)
- **sigma-rust**: Rust implementation (ergoplatform/sigma-rust)
- **Ergo node**: Full node implementation showing integration
- **Formal specifications**: LaTeX documents in `docs/spec/`
- **Test suites**: Language specification tests defining expected behavior

Citations use footnotes referencing both Scala and Rust source locations.

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

```zig
// Code blocks use Zig to illustrate algorithms
const ErgoTree = struct {
    header: Header,
    constants: []const Constant,
    root: *const Expr,
};
```

> **Note**: Highlighted notes provide important context or warnings.

**Footnotes**: `[^1]: Scala: path/to/file.scala:123` and `[^2]: Rust: path/to/file.rs:456` reference source locations in both implementations.

## Version Information

This book documents:
- **sigmastate-interpreter**: Version 6.x (with notes on v5 differences)
- **sigma-rust**: ergotree-ir and ergotree-interpreter crates
- **Protocol versions**: v0 (initial), v1 (v4.0), v2 (v5.0 JIT), v3 (v6.0)

## Contributing

This book is maintained as part of the ErgoTree research project. Corrections and improvements are welcome.

---

*Let's begin with [Chapter 1: Introduction to Sigma and ErgoTree](./part1/ch01-introduction.md).*
