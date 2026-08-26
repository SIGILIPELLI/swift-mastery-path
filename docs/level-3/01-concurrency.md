# 01 · Concurrency

Swift's structured concurrency (`async`/`await`, `Task`, actors) replaces the
old world of completion handlers and manual queue-hopping with code that
reads top-to-bottom but still runs cooperatively. This module covers the
core building blocks you'll use in every concurrent Swift program from here
on.

## `async` functions and `await`

A function marked `async` can suspend without blocking the thread it's
running on. You call it with `await`, which marks the point where suspension
may happen:

```swift
import Foundation

func fetchNumber(_ n: Int) async -> Int {
    try? await Task.sleep(nanoseconds: 100_000_000)   // simulate 0.1s of work
    return n * n
}

func sumAll() async -> Int {
    var total = 0
    for n in 1...3 {
        total += await fetchNumber(n)   // each call waits for the previous one
    }
    return total
}
```

`Task.sleep` suspends the current task without blocking the thread — other
work can run on that thread while this call waits.

## Running work concurrently with `async let`

`sumAll` above runs its three calls one after another. `async let` starts
several `async` calls at once and lets you `await` their results later:

```swift
func sumConcurrently() async -> Int {
    async let a = fetchNumber(1)
    async let b = fetchNumber(2)
    async let c = fetchNumber(3)
    return await a + b + c   // all three run in parallel, then we wait for all
}
```

Both functions compute the same total, but `sumConcurrently` finishes in
roughly the time of one `fetchNumber` call instead of three, because the
sleeps overlap.

## `Task`: launching unstructured work

`Task { ... }` starts a new unit of async work you can hold onto and `await`
later:

```swift
let task = Task {
    try? await Task.sleep(nanoseconds: 50_000_000)
    return "task result"
}
let result = await task.value
print("Task result:", result)
```

`Task` inherits the priority and (in structured contexts) the cancellation
state of whatever created it, which is why it's called "unstructured but
still connected" concurrency — it's not tied to a parent scope the way
`async let` and task groups are, but it isn't fully detached either.

## Task groups: a dynamic number of children

`async let` needs a fixed number of concurrent calls written out by hand.
`withTaskGroup` lets you fan out a *dynamic* number of child tasks and
collect their results:

```swift
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 {
        group.addTask { await counter.increment() }
    }
}
```

This adds 100 child tasks to the group; the `await` on the group's closure
returns only once every child has finished.

## Actors: shared mutable state without data races

Ordinary classes shared across concurrent tasks are a data-race hazard —
two tasks can read-modify-write the same property at once and corrupt it.
An `actor` protects its state by only ever running one piece of its own code
at a time, isolating access automatically:

```swift
actor Counter {
    private var value = 0
    func increment() { value += 1 }
    var current: Int { value }
}

func runActorDemo() async {
    let counter = Counter()
    await withTaskGroup(of: Void.self) { group in
        for _ in 0..<100 {
            group.addTask { await counter.increment() }
        }
    }
    let final = await counter.current
    print("Counter after 100 concurrent increments:", final)
}
```

Every access to `counter` from outside the actor needs `await`, even though
`increment()` itself does no explicit async work — that `await` is the
compiler enforcing that you might have to wait for the actor to be free.

## Full runnable example and output

Putting it all together (top-level `await` works directly in a `swift`
script file — no `@main` needed there):

```swift
let sequential = await sumAll()
print("Sequential sum:", sequential)

let concurrent = await sumConcurrently()
print("Concurrent sum:", concurrent)

await runActorDemo()

let task = Task {
    try? await Task.sleep(nanoseconds: 50_000_000)
    return "task result"
}
let result = await task.value
print("Task result:", result)
```

Running it with `swift concurrency.swift`:

```
Sequential sum: 14
Concurrent sum: 14
Counter after 100 concurrent increments: 100
Task result: task result
```

Both sums are `14` (1 + 4 + 9) — concurrency changes *when* work overlaps,
not *what* it computes. The counter example is the important one: without
the actor, 100 concurrent increments on a plain class would frequently land
on a number less than 100 due to lost updates; the actor guarantees exactly
100 every time.

## Swift-specific traps

- **`@main` and top-level code don't mix.** A file with top-level statements
  (like a `.swift` script) can't also declare a `@main` type — pick one. In
  an SPM executable target with no other top-level code, `@main struct App`
  with a `static func main() async` works fine.
- **`await` doesn't mean "runs on a new thread."** It means "may suspend
  here." Cheap `async` functions may never actually suspend, and suspension
  doesn't guarantee a thread switch either — don't reason about concurrency
  in terms of threads.
- **Forgetting `await` on actor access is a compile error, not a runtime
  bug** — Swift's actor isolation is checked at compile time, which is a
  feature: you can't accidentally skip the safety.
- **`async let` bindings are eagerly started but must be awaited (or
  cancelled) before they go out of scope**, otherwise Swift implicitly
  awaits and cancels them for you when the enclosing scope exits — relying
  on that instead of an explicit `await` makes control flow harder to read.

## Cheat sheet

| Tool | Use it when |
|------|-------------|
| `async`/`await` | Marking and calling a function that may suspend |
| `async let` | A fixed, known number of concurrent child computations |
| `Task { }` | Starting unstructured async work from sync code |
| `withTaskGroup` | A dynamic number of concurrent child tasks |
| `actor` | Protecting mutable state shared across concurrent tasks |
| `Task.sleep(nanoseconds:)` | Non-blocking delay inside async code |

## Exercise

Write an `actor BankAccount` with a private `balance: Int` starting at `0`
and a method `func deposit(_ amount: Int)`. Using `withTaskGroup`, launch 50
concurrent tasks that each deposit `10`. After the group completes, read and
print the final balance through an `await`ed computed property — confirm it
is exactly `500`, proving the actor prevented lost updates.
