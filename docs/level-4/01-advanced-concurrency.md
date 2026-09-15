---
description: "Advanced Concurrency — Level 3 covered async/await, Task, async let, and actors. This module goes further: streaming values over time with AsyncSequence…"
---

# 01 · Advanced Concurrency

Level 3 covered `async`/`await`, `Task`, `async let`, and actors. This
module goes further: streaming values over time with `AsyncSequence`,
cooperative cancellation, and structured error propagation through task
groups.

## `AsyncStream`: bridging callback-style event sources

`AsyncStream` turns a push-based source (a callback, a delegate, a timer)
into something you can `for await` over:

```swift
import Foundation

func countdownStream(from start: Int) -> AsyncStream<Int> {
    AsyncStream { continuation in
        Task {
            var current = start
            while current >= 0 {
                continuation.yield(current)
                try? await Task.sleep(nanoseconds: 10_000_000)
                current -= 1
            }
            continuation.finish()
        }
    }
}

var streamValues: [Int] = []
for await value in countdownStream(from: 3) {
    streamValues.append(value)
}
print("Countdown stream:", streamValues)
```

Output:

```
Countdown stream: [3, 2, 1, 0]
```

`continuation.yield` pushes a value to whoever is iterating the stream;
`continuation.finish()` ends the `for await` loop, the same way returning
`nil` ends a regular `IteratorProtocol` sequence.

## Writing a custom `AsyncSequence`

For a computed sequence (rather than a bridged callback source), conform
directly to `AsyncSequence` and `AsyncIteratorProtocol`:

```swift
struct Fibonacci: AsyncSequence {
    typealias Element = Int
    let count: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        var a = 0, b = 1
        var remaining: Int
        mutating func next() async -> Int? {
            guard remaining > 0 else { return nil }
            remaining -= 1
            let value = a
            (a, b) = (b, a + b)
            return value
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(remaining: count)
    }
}

var fibValues: [Int] = []
for await value in Fibonacci(count: 8) {
    fibValues.append(value)
}
print("Fibonacci:", fibValues)
```

Output:

```
Fibonacci: [0, 1, 1, 2, 3, 5, 8, 13]
```

This looks almost identical to a synchronous custom `Sequence` — the only
difference is `next()` is `async`, which is what lets each step suspend
(for I/O, a timer, etc.) without blocking a thread.

## Cooperative cancellation

Swift concurrency cancellation is cooperative: calling `task.cancel()`
doesn't forcibly stop anything — it flips a flag that running code must
check (`Task.isCancelled`) and choose to respect:

```swift
func longRunningTask() async -> String {
    for i in 0..<10 {
        if Task.isCancelled {
            return "cancelled at step \(i)"
        }
        try? await Task.sleep(nanoseconds: 20_000_000)
    }
    return "completed"
}

let task = Task {
    await longRunningTask()
}
try? await Task.sleep(nanoseconds: 50_000_000)
task.cancel()
let result = await task.value
print("Cancellation result:", result)
```

Output (exact step number depends on scheduling, but the pattern holds):

```
Cancellation result: cancelled at step 3
```

`Task.sleep` itself throws `CancellationError` when the task it's running
in is cancelled, which is why real code often uses `try await Task.sleep`
(propagating the error) rather than `try?` (silently swallowing it) —
this example swallows it deliberately to keep looping and checking
`isCancelled` explicitly instead.

## Error propagation through `withThrowingTaskGroup`

A throwing task group cancels its remaining sibling tasks the moment any
child throws, and the error surfaces from the `for try await` loop or the
group's return:

```swift
enum WorkError: Error { case failed(Int) }

func riskyWork(_ n: Int) async throws -> Int {
    if n == 3 { throw WorkError.failed(n) }
    return n * n
}

func runGroupWithErrors() async {
    do {
        let results = try await withThrowingTaskGroup(of: Int.self) { group -> [Int] in
            for n in 1...5 {
                group.addTask { try await riskyWork(n) }
            }
            var values: [Int] = []
            for try await value in group {
                values.append(value)
            }
            return values
        }
        print("Results:", results)
    } catch {
        print("Group failed with error:", error)
    }
}

await runGroupWithErrors()
```

Output:

```
Group failed with error: failed(3)
```

Note that `Results:` never prints — the moment the child processing `n ==
3` throws, `withThrowingTaskGroup` propagates that error out of the whole
`do` block, and the other four children are cancelled (though any that
already completed have their results discarded here since we never got to
use `values`).

## Swift-specific traps

- **Cancellation is advisory, not automatic** — an `async` function that
  never checks `Task.isCancelled` or calls a cancellation-aware API (like
  `Task.sleep`, which throws) will run to completion regardless of
  `cancel()` having been called on it.
- **`try?` around `Task.sleep` hides `CancellationError`**, which is
  sometimes exactly what you want (as in the loop above, to keep checking
  the flag) and sometimes a bug (silently continuing work that should have
  stopped) — be deliberate about which one you mean.
- **`AsyncStream`'s closure runs the producing `Task` even if nobody is
  iterating yet** in some construction patterns — for expensive producers,
  make sure the `Task` only starts once the stream actually has a
  consumer, or use the buffering policy parameter to bound memory use.
- **A throwing task group's `addTask` closures keep running until they
  reach their own next suspension point**, even after a sibling has
  thrown — cancellation is checked, not enforced instantly, so a child deep
  in synchronous work won't stop mid-loop unless it explicitly checks
  `Task.isCancelled`.

## Cheat sheet

| Tool | Use it for |
|------|------------|
| `AsyncStream` | Bridging a callback/delegate source into `for await` |
| Custom `AsyncSequence` | A computed sequence of values produced asynchronously |
| `Task.isCancelled` | Checking cancellation cooperatively inside a loop |
| `Task.sleep` (thrown, not `try?`ed) | Propagating cancellation automatically during a wait |
| `withThrowingTaskGroup` | Fanning out children where any failure should abort the group |

## How It Actually Works

- **`TaskGroup` builds an explicit parent/child task tree** at runtime: adding a
  child task (`group.addTask { }`) registers it with the group's underlying
  task record, and the group's `for await` iteration polls each child's result
  as it completes — critically, if the group's own task is cancelled or exits
  its scope (including via a thrown error), the runtime automatically walks
  this tree and propagates cancellation to every still-running child, which is
  the actual mechanism behind "structured concurrency guarantees no task
  outlives its scope."
- **Cancellation is cooperative, implemented as a checked flag, not a forced
  interrupt.** Calling `task.cancel()` flips an atomic flag on that task's
  shared state; nothing stops automatically. Code must explicitly check
  `Task.isCancelled` or call `try Task.checkCancellation()` (which throws
  `CancellationError`) at appropriate points — a tight non-async loop that
  never checks cancellation will keep running forever regardless of being
  "cancelled."
- **Custom executors and `AsyncSequence`** — implementing an executor means
  providing the low-level scheduling primitive Swift Concurrency's runtime
  calls into to actually run a job (typically wrapping your own thread/queue).
  `AsyncSequence`'s `for await` loop compiles to repeated calls to
  `makeAsyncIterator()`/`next()`, each an `async` call that can genuinely
  suspend the consuming task until the next element becomes available — the
  same state-machine transformation underlying every `async` function applies
  here too, just invoked in a loop.
- **`@MainActor` isolation** is enforced identically to any other actor
  isolation (see the concurrency chapter) — code marked `@MainActor` runs on a
  specific global-actor-owned serial executor that happens to be backed by the
  main thread/run loop, and calling it from off-main-thread code requires an
  `await` hop that the compiler statically enforces, catching "touched UI from
  a background thread" bugs at compile time instead of as an intermittent
  crash.

## Exercise

Write an `AsyncSequence` called `Paginated<Element>` that simulates fetching
"pages" of data from a server: given a total item count and a page size, its
`next()` should `await Task.sleep` briefly (simulating network latency),
then return one item at a time, `nil` once exhausted. Iterate it with `for
await` and print every item along with an index. Then wrap the iteration in
a `Task`, cancel it after receiving the third item, and confirm your
sequence's `next()` checks `Task.isCancelled` and stops early instead of
running to completion.
