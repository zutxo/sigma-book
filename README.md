# The Sigma Book

[![mdBook](https://img.shields.io/badge/mdBook-Documentation-blue)](https://rust-lang.github.io/mdBook/)

**Complete Technical Reference for SigmaState Interpreter, ErgoTrees, and Sigma Language**

A comprehensive technical reference for engineers who need deep understanding of the implementation details, algorithms, and design decisions behind the Ergo blockchain's smart contract system.

## Contents

The book is organized into ten parts covering the full stack:

| Part | Title | Chapters |
|------|-------|----------|
| I | **Foundations** | Introduction, Type System, ErgoTree Structure |
| II | **Abstract Syntax Tree** | Value Nodes, Operations & Opcodes, Methods on Types |
| III | **Serialization** | Serialization Framework, Value Serializers |
| IV | **Cryptographic Foundations** | Elliptic Curve Cryptography, Hash Functions, Sigma Protocols |
| V | **Interpreter Engine** | Evaluation Model, Cost Model, Verifier, Prover |
| VI | **Compiler** | ErgoScript Parser, Semantic Analysis, IR, Pipeline |
| VII | **Data Structures** | Collections, AVL+ Trees, Box Model |
| VIII | **Ergo Node Integration** | Interpreter Wrappers, Transaction Validation, Cost Limits, Wallet |
| IX | **SDK and APIs** | High-Level SDK, Key Derivation |
| X | **Advanced Topics** | Soft-Fork Mechanism, Cross-Platform, Performance |

Plus comprehensive appendices covering type codes, opcodes, costs, methods, serialization formats, and version history.

## Building the Book

### Prerequisites

- [Rust](https://rustup.rs/) (for installing mdBook)
- [mdBook](https://rust-lang.github.io/mdBook/)

### Install mdBook

```bash
cargo install mdbook
```

### Build and Serve

```bash
# Build the book
mdbook build

# Serve locally with hot reload
mdbook serve --open
```

The built book will be in the `book/` directory.

## Source Material

This book is derived from:

- **sigmastate-interpreter** — The reference Scala implementation
- **Ergo node** — Full node implementation showing integration
- **Formal specifications** — LaTeX documents and academic papers
- **Test suites** — Comprehensive tests defining expected behavior

## Contributing

Contributions are welcome. Please open an issue or pull request on GitHub.

## License

See [LICENSE](LICENSE) for details.
