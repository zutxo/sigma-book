# Chapter 31: Performance Engineering

## Prerequisites

- **Required knowledge**: Scala programming, JVM runtime characteristics
- **Related concepts**: Interpreter evaluation (Chapter 12), Serialization (Chapter 7)
- **Prior chapters**: Understanding of core interpreter and compiler components

## Learning Objectives

- Understand performance-critical patterns in the codebase
- Learn the HOTSPOT annotation convention and its implications
- Master memoization strategies for expensive operations
- Apply collection optimization techniques
- Understand the cfor macro for efficient looping

## Source References

- `docs/perf-style-guide.md` (comprehensive performance guide)
- `core/shared/src/main/scala/sigma/util/MemoizedFunc.scala`
- `core/shared/src/main/scala/sigma/kiama/rewriting/Rewriter.scala`

## Introduction

SigmaState processes thousands of transactions per block, each requiring script deserialization and verification. Performance is critical—every unnecessary allocation or inefficient loop multiplies across the network. This chapter documents the performance engineering practices that keep the interpreter fast.

The codebase uses **HOTSPOT** annotations to mark performance-critical code that should not be "beautified" (refactored for readability at the expense of speed). Understanding these patterns is essential for maintaining and extending the interpreter.

## The HOTSPOT Convention

Throughout the codebase, you'll find comments like:

```scala
// HOTSPOT: don't beautify this code
// HOTSPOT: executed on every typeCode during deserialization
// HOTSPOT:: avoids thousands of allocations per second
```

These markers indicate code paths executed frequently—during deserialization, evaluation, or proof generation. The convention serves two purposes:

1. **Warning to maintainers**: Don't refactor this code for style without measuring performance impact
2. **Documentation**: Explains why code might look "ugly" compared to idiomatic Scala

### Where HOTSPOTs Occur

HOTSPOT annotations appear in:

| Component | Example Files | Frequency |
|-----------|--------------|-----------|
| Serialization | `ErgoTreeSerializer`, `ValueSerializer` | Per-tree |
| Evaluation | `CErgoTreeEvaluator`, `fixedCostOp` | Per-operation |
| Proof handling | `SigSerializer`, `UnprovenTree` | Per-proof |
| Validation | `ValidationRules`, `checkRule` | Per-node |
| IR transforms | `GraphBuilding`, `Transforming` | Per-compile |

### HOTSPOT Code Characteristics

HOTSPOT code typically exhibits:

```scala
// From ErgoTreeSerializer.scala:243-256
// HOTSPOT: don't beautify this code
private def deserializeConstants(header: HeaderType, r: SigmaByteReader): IndexedSeq[Constant[SType]] = {
  val constants: IndexedSeq[Constant[SType]] =
    if (ErgoTree.hasSize(header)) {
      val nConsts = r.getUInt().toInt
      if (nConsts > 0) {
        // HOTSPOT:: allocate new array only if it is not empty
        val res = safeNewArray[Constant[SType]](nConsts)
        cfor(0)(_ < nConsts, _ + 1) { i =>
          res(i) = constantSerializer.deserialize(r)
        }
        res
      } else Constant.EmptySeq
    } else Constant.EmptySeq
  constants
}
```

Key patterns visible here:
- **Conditional allocation**: Only create array when needed
- **`cfor` macro**: Efficient iteration without closure allocation
- **Pre-allocated empty sequences**: Avoid creating new empty collections
- **Direct array manipulation**: Mutable arrays over immutable collections

## The cfor Macro

Standard Scala `for` loops are convenient but expensive:

```scala
// This innocuous code...
for (x <- xs) { process(x) }

// ...desugars to:
xs.foreach { x => process(x) }

// Which involves:
// 1. Method call to foreach
// 2. Lambda object allocation
// 3. Boxing of primitive arguments
// 4. Hidden iterator overhead
```

### cfor Solution

The `cfor` macro from the debox library compiles to efficient Java `for` loops:

```scala
import debox.cfor

// Instead of: for (i <- 0 until n) { ... }
cfor(0)(_ < n, _ + 1) { i =>
  val x = xs(i)
  process(x)
}
```

This is **20-50x faster** than `foreach` for performance-critical loops.

### cfor Usage Pattern

```scala
// From Transforming.scala:247-253
// HOTSPOT:
def mirrorSymbols(t0: Transformer, rewriter: Rewriter, nodes: DBuffer[Int]): Transformer = {
  var t: Transformer = t0
  cfor(0)(_ < nodes.length, _ + 1) { i =>
    val id = nodes(i)
    t = mirrorNode(t, rewriter, getSym(id))
  }
  t
}
```

The `cfor` call sites in the codebase include:
- `GF2_192.scala` and `GF2_192_Poly.scala` - Cryptographic operations
- `GraphBuilding.scala` - IR construction
- `AstGraphs.scala` - Schedule building
- `ErgoTreeSerializer.scala` - Constant deserialization
- Various serializers throughout

## Memoization

Expensive operations that produce deterministic results should be computed once:

### MemoizedFunc Class

```scala
// From MemoizedFunc.scala
class MemoizedFunc(f: AnyRef => AnyRef) {
  private var _table: AVHashMap[AnyRef, AnyRef] = AVHashMap(100)

  /** Apply function using cached result if available. */
  def apply[T <: AnyRef](x: T): AnyRef = {
    var v = _table(x)
    if (v == null) {
      v = f(x)
      _table.put(x, v)
    }
    v
  }

  /** Clears the cache of memoized results. */
  def reset() = {
    _table = AVHashMap(100)
  }
}
```

Key design choices:
- **`null` vs `Option`**: Using `null` avoids `Option` allocation on every lookup
- **AVHashMap**: Custom hash map optimized for the use case
- **Initial capacity 100**: Pre-sized for typical usage patterns
- **Reset capability**: Clear cache between independent operations

### When to Memoize

Good candidates for memoization:
- Type checking results
- Method resolution
- Cost calculations for repeated structures
- Hash computations

Bad candidates:
- Functions with side effects
- Functions with mutable arguments
- Operations already fast enough
- Operations called only once

## Collection Optimization

### Empty Collections

Creating empty collections has overhead:

```scala
// Slow: calls apply method, checks isEmpty, creates builder
Seq()
Map()

// Fast: returns pre-allocated singleton
Nil           // 3-20x faster than Seq()
Map.empty     // 50-70% faster than Map()
```

The codebase defines pre-allocated empty sequences:

```scala
// From Constant.scala
object Constant {
  val EmptySeq: IndexedSeq[Constant[SType]] = Array.empty[Constant[SType]]
}

// From ContractTemplate.scala
val EmptySeq: IndexedSeq[Parameter] = Array.empty[Parameter]
```

### Array vs Seq

When you need a fixed-size collection:

```scala
// Slow: multiple allocations, boxing, list construction
Seq(1, 2, 3)

// Fast: single array allocation
Array(1, 2, 3)  // 4-10x faster
```

Arrays wrapped implicitly provide `Seq` interface:

```scala
// From transformers.scala:502-503
// HOTSPOT:: avoids thousands of allocations per second
private val BoxAndByte: IndexedSeq[SType] = Array(SBox, SByte)
```

This pattern:
1. Creates a single array at class loading time
2. Implicitly converts to `IndexedSeq` when needed
3. Avoids repeated allocations in hot paths

### Pre-allocated Arrays

For frequently accessed collections:

```scala
// From ErgoBox.scala:174-176
// HOTSPOT: don't beautify the code in this companion
private val _mandatoryRegisters: Array[MandatoryRegisterId] = Array(R0, R1, R2, R3)
val mandatoryRegisters: Seq[MandatoryRegisterId] = _mandatoryRegisters
```

## Avoiding Allocations

### null vs Option

In HOTSPOTs, `null` is often preferred over `Option`:

```scala
// From CErgoTreeEvaluator.scala:487-489
// HOTSPOT: don't beautify the code
// Note, `null` is used instead of Option to avoid allocations.
def fixedCostOp[R](costInfo: OperationCostInfo[FixedCost])
                  (block: => R)(implicit E: ErgoTreeEvaluator): R
```

The trade-off:
- `Option`: Safe, idiomatic, but allocates `Some()` wrapper
- `null`: Unsafe (NPE risk), but zero allocation overhead

This is acceptable in HOTSPOTs because:
1. The code paths are well-tested
2. The performance gain is measurable
3. The scope is limited and clearly documented

### Value Classes and Extractors

Standard pattern matching allocates on extraction:

```scala
object PositiveInt {
  def unapply(n: Int): Option[Int] =
    if (n > 0) Some(n) else None  // Allocates Some every match
}
```

Name-based extractors with value classes avoid this:

```scala
// Uses spire.util.Opt or similar value class pattern
class Opt[+A](val value: A) extends AnyVal {
  def isEmpty: Boolean = value == null
  def get: A = value
}

object PositiveInt {
  def unapply(n: Int): Opt[Int] =
    if (n > 0) new Opt(n) else Opt.empty  // No allocation (value class)
}
```

This is **1.5-2x faster** than Option-based extractors.

## Abstract Class vs Trait

Method invocation differs by type:

```scala
trait Foo {
  def method(): Int  // Uses invokeinterface
}

abstract class Bar {
  def method(): Int  // Uses invokevirtual
}
```

`invokeinterface` is slower because:
1. JIT has harder time optimizing
2. Method lookup requires additional indirection
3. Less predictable call sites hinder inlining

Recommendation: Use `abstract class` by default, `trait` only when mixin composition is required.

## Lazy Initialization

For expensive computations not always needed:

```scala
// From SigmaByteReader.scala:27-32
// The reader should be lightweight to create. In most cases ErgoTrees don't have
// ValDef nodes hence the store is not necessary and it's initialization dominates
// the reader instantiation time. Hence it's lazy.
// HOTSPOT:
lazy val valDefTypeStore: ValDefTypeStore = new ValDefTypeStore()
```

Use `lazy val` when:
- Initialization is expensive
- Value may not be needed in all code paths
- Value is needed multiple times if used at all

Avoid `lazy val` when:
- Initialization is cheap (overhead of lazy outweighs benefit)
- Value is always needed
- Thread contention is a concern (lazy val has synchronization overhead)

## Validation Rule Optimization

Validation rules are checked frequently:

```scala
// From ValidationRules.scala:24-30
// Check the rule is registered and enabled.
// Since it is easy to forget to register new rule, we need to do this check.
// But because it is hotspot, we do this check only once for each rule.
// HOTSPOT: executed on every typeCode and opCode during script deserialization
@inline protected final def checkRule(): Unit = {
  if (!_checked) {
    // First-time check: verify registration
    _checked = true
  }
}
```

Pattern: **First-time initialization** with flag check is faster than repeated lookups or registration verification.

## Tree Rewriting Optimization

The compiler uses Stratego-style term rewriting:

```scala
// From Rewriter.scala
trait Rewriter {
  def rewrite[T](s: Strategy)(t: T): T = {
    s(t) match {
      case Some(t1) => t1.asInstanceOf[T]
      case None     => t
    }
  }
}
```

Optimizations in graph rewriting:

```scala
// From GraphBuilding.scala:154-158
// Unfortunately, this is less readable, but gives significant performance boost
// Look at comments to understand the logic of the rules.
// HOTSPOT: executed for each node of the graph, don't beautify.
override def rewriteDef[T](d: Def[T]): Ref[_] = {
  // Direct pattern matching without intermediate abstractions
}
```

The `rewriteDef` method uses extensive pattern matching directly rather than through helper methods, trading readability for speed.

## Benchmarking

The codebase includes benchmark suites:

```
sc/jvm/src/test/scala/special/collections/
├── BasicBenchmarks.scala
├── BufferBenchmark.scala
├── CollBenchmark.scala
├── MapBenchmark.scala
└── SymBenchmark.scala
```

### Benchmarking Principles

1. **Isolate**: Benchmark single operations, not whole pipelines
2. **Warm up**: Allow JIT compilation before measuring
3. **Repeat**: Run many iterations to reduce variance
4. **Compare**: Always benchmark against baseline
5. **Profile**: Use JITWatch or async-profiler for deeper analysis

### Example Benchmark Pattern

```scala
def benchmark[T](name: String, iterations: Int)(block: => T): Unit = {
  // Warm up
  cfor(0)(_ < 1000, _ + 1) { _ => block }

  // Measure
  val start = System.nanoTime()
  cfor(0)(_ < iterations, _ + 1) { _ => block }
  val elapsed = System.nanoTime() - start

  println(s"$name: ${elapsed / iterations} ns/op")
}
```

## Performance Checklist

When writing performance-critical code:

- [ ] Use `cfor` instead of `for` loops on indexed collections
- [ ] Use `Nil` instead of `Seq()` for empty sequences
- [ ] Use `Map.empty` instead of `Map()` for empty maps
- [ ] Use `Array(...)` instead of `Seq(...)` for small fixed collections
- [ ] Pre-allocate frequently used empty collections
- [ ] Consider `null` instead of `Option` in true HOTSPOTs
- [ ] Use `abstract class` instead of `trait` where possible
- [ ] Add HOTSPOT comment to document performance-critical code
- [ ] Benchmark before and after changes
- [ ] Consider memoization for expensive pure functions

## Exercises

1. **Conceptual**: Why does `Seq()` perform worse than `Nil`? Trace through the Scala standard library to understand the allocation differences.

2. **Measurement**: Write a benchmark comparing `for (i <- 0 until n)` vs `cfor(0)(_ < n, _ + 1)` for different values of n. At what n does the difference become significant?

3. **Analysis**: Find three HOTSPOT annotations in the codebase and explain what makes each code path performance-critical.

4. **Implementation**: Implement a memoized version of a recursive Fibonacci function using the `MemoizedFunc` pattern. Compare performance with the naive recursive version.

## Summary

- **HOTSPOT annotations** mark performance-critical code that shouldn't be refactored for style
- **cfor macro** provides 20-50x faster loops than standard Scala `for`
- **Memoization** caches expensive computations with `MemoizedFunc`
- **Collection optimization**: Use `Nil`, `Map.empty`, `Array(...)` over generic constructors
- **Allocation avoidance**: Use `null` over `Option` in true HOTSPOTs, pre-allocate empty collections
- **Abstract class over trait** for faster method dispatch
- **Benchmark everything** before and after performance changes

## Further Reading

- Source: `docs/perf-style-guide.md` (complete performance guide)
- Source: `core/shared/src/main/scala/sigma/util/MemoizedFunc.scala`
- [Scala High Performance Programming](https://www.amazon.com/Scala-Performance-Programming-Vincent-Theron/dp/178646604X)
- [JITWatch](https://github.com/AdoptOpenJDK/jitwatch) - JIT compilation visualizer
- [debox library](https://github.com/ScorexFoundation/debox) - High-performance Scala collections

---
*[Previous: Chapter 30](./ch30-cross-platform-support.md) | [Next: Appendix A](../appendices/appendix-a-type-codes.md)*
