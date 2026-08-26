# 07 · Performance at Scale

Level 3's performance module covered per-call micro-optimizations
(`reserveCapacity`, `Set` vs `Array`). This module is about performance
*at scale* — the techniques that matter once you're processing thousands
to millions of items or serving concurrent requests: memoization, batching
(and when it doesn't help), and parallelizing independent work.

## A timing helper

Reusing the small timer from Level 3's performance module:

```swift
import Foundation

func measure(_ label: String, _ block: () -> Void) {
    let start = DispatchTime.now()
    block()
    let end = DispatchTime.now()
    let ms = Double(end.uptimeNanoseconds - start.uptimeNanoseconds) / 1_000_000
    print(String(format: "%@: %.2fms", label, ms))
}
```

## Memoization: trading memory for repeated work

Naive recursive Fibonacci recomputes the same sub-results exponentially
many times. Caching each result the first time it's computed collapses
that into linear work:

```swift
final class Memoizer {
    private var cache: [Int: Int] = [:]
    private var calls = 0

    func fib(_ n: Int) -> Int {
        calls += 1
        if let cached = cache[n] { return cached }
        let result = n < 2 ? n : fib(n - 1) + fib(n - 2)
        cache[n] = result
        return result
    }

    var callCount: Int { calls }
}

let memo = Memoizer()
let result = memo.fib(30)
print("fib(30) =", result, "computed with", memo.callCount, "calls (memoized)")
```

Output:

```
fib(30) = 832040 computed with 59 calls (memoized)
```

59 calls total for `fib(30)` — compare to the naive version, which makes
roughly 2.7 million recursive calls for the same input, because it
recomputes `fib(28)`, `fib(27)`, etc. from scratch every time they're
needed again.

## Measuring the difference

```swift
func fibNaive(_ n: Int) -> Int {
    n < 2 ? n : fibNaive(n - 1) + fibNaive(n - 2)
}

measure("naive fib(28)") { _ = fibNaive(28) }

let memo2 = Memoizer()
measure("memoized fib(28)") { _ = memo2.fib(28) }
```

Output:

```
naive fib(28): 1.30ms
memoized fib(28): 0.01ms
```

Two orders of magnitude faster at `n = 28` — and the gap widens
exponentially as `n` grows, since the naive version's work grows
exponentially while the memoized version's stays linear.

## Batching: helps when overhead is per-call, not per-item

A common claim is "batch your work for better performance" — but that's
only true when there's a fixed cost *per call* that batching amortizes
(a network round-trip, a database query, a lock acquisition). For pure
in-memory CPU work with no such per-call overhead, batching can actually
add cost from the extra bookkeeping:

```swift
func expensiveStep(_ x: Int) -> Int { x * x }

func processOneByOne(_ items: [Int]) -> Int {
    var total = 0
    for item in items { total += expensiveStep(item) }
    return total
}

func processBatched(_ items: [Int], batchSize: Int) -> Int {
    var total = 0
    var index = 0
    while index < items.count {
        let end = min(index + batchSize, items.count)
        let batch = items[index..<end]
        total += batch.reduce(0) { $0 + expensiveStep($1) }
        index = end
    }
    return total
}

let items = Array(0..<100_000)
measure("one by one") { _ = processOneByOne(items) }
measure("batched") { _ = processBatched(items, batchSize: 1000) }
```

Output on this machine:

```
one by one: 7.08ms
batched: 11.16ms
```

The batched version is *slower* here — there's no per-call overhead to
amortize (`expensiveStep` is a trivial in-process function call), so the
extra array slicing and closure-based `reduce` per batch is pure added
cost. **Batching earns its keep specifically when each unbatched operation
has a real fixed cost** — a network request, a `SELECT`/`INSERT`
round-trip to a database, an actor hop — not for tight in-memory loops.
Measure before assuming; the "obviously correct" optimization here would
have made things worse.

## Parallelizing independent chunks of work

Where batching didn't help, splitting genuinely independent CPU work across
concurrent tasks can — each `withTaskGroup` child can run on a different
CPU core:

```swift
func sumOfSquares(_ range: Range<Int>) -> Int {
    range.reduce(0) { $0 + $1 * $1 }
}

func parallelSum(_ total: Int, chunks: Int) async -> Int {
    let chunkSize = total / chunks
    return await withTaskGroup(of: Int.self) { group -> Int in
        for i in 0..<chunks {
            let start = i * chunkSize
            let end = (i == chunks - 1) ? total : start + chunkSize
            group.addTask { sumOfSquares(start..<end) }
        }
        var sum = 0
        for await partial in group { sum += partial }
        return sum
    }
}

let parallelResult = await parallelSum(1_000_000, chunks: 4)
let sequentialResult = sumOfSquares(0..<1_000_000)
print("Parallel result matches sequential:", parallelResult == sequentialResult)
```

Output:

```
Parallel result matches sequential: true
```

Splitting `0..<1_000_000` into 4 non-overlapping ranges and summing each
independently, then combining, is correct here because summation is
associative — the chunks don't need to communicate. This pattern
(partition → process independently → combine) is the shape of most
"embarrassingly parallel" performance wins; work with dependencies between
chunks (like the Fibonacci memoization above) doesn't parallelize this
cleanly.

## Swift-specific traps

- **Don't assume batching, chunking, or parallelizing helps — measure.**
  As shown above, the same-looking "obvious optimization" made things worse
  for pure CPU work with no per-call overhead.
- **`withTaskGroup` overhead is real** for very cheap per-chunk work —
  spinning up child tasks has its own cost, so parallelizing a
  microsecond-scale computation can lose to just doing it sequentially;
  the technique pays off once each chunk's work meaningfully exceeds task
  overhead.
- **Memoization caches grow unbounded by default** — the `Memoizer` above
  never evicts anything; a production cache needs an eviction policy (LRU,
  a size cap, or a TTL) or it becomes its own memory leak under sustained
  load.
- **Recursive memoization with a class-based cache is not
  concurrency-safe** — `Memoizer` above uses a plain `class`; called from
  multiple concurrent tasks, its `cache` dictionary would be a data race.
  Wrapping it as an `actor` (Level 3, Module 01) fixes that at the cost of
  requiring `await` at every call site.

## Cheat sheet

| Technique | Helps when | Doesn't help when |
|-----------|-----------|--------------------|
| Memoization | Overlapping/repeated sub-computations (recursion, shared queries) | Every input is unique — nothing to reuse |
| Batching | Per-call overhead is real (network, DB, actor hops) | Pure in-memory CPU work with no per-call cost |
| `withTaskGroup` parallelism | Chunks are independent and individually substantial | Chunks are tiny, or depend on each other's results |

## Exercise

Take the `processOneByOne`/`processBatched` comparison above and change
`expensiveStep` to simulate a real per-call cost — for example, `try?
await Task.sleep(nanoseconds: 100_000)` inside an `async` version reached
one item at a time versus fetched via a single batched
`withTaskGroup`-based fan-out. Measure both with the `measure` helper
(adapted for `async` code) and confirm that *this* time, batching (or
concurrent fan-out) wins — because now there's real per-call overhead to
amortize, unlike the pure-CPU case in this module.
