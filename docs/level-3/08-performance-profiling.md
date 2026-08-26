# 08 · Performance & Profiling

Most performance problems in Swift come from a handful of repeat offenders:
unnecessary array growth, string concatenation in a loop, and using the
wrong collection for a lookup-heavy workload. This module measures each one
with a small hand-rolled timer, then covers Instruments — the real
profiling tool — for when guessing isn't good enough.

## A simple timing helper

`DispatchTime.now()` gives nanosecond-resolution timestamps without any
external dependency:

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

## Trap 1: growing an array without `reserveCapacity`

`Array` grows by doubling its capacity whenever it runs out — cheap on
average, but each doubling still copies existing elements into new storage.
Telling it the final size upfront skips those intermediate reallocations:

```swift
let count = 200_000

measure("append without reserve") {
    var array: [Int] = []
    for i in 0..<count { array.append(i) }
}

measure("append with reserveCapacity") {
    var array: [Int] = []
    array.reserveCapacity(count)
    for i in 0..<count { array.append(i) }
}
```

## Trap 2: `String` concatenation with `+=` in a loop

`String += String` can look innocent but repeatedly bridges/copies
depending on the strings involved; building an array of pieces and joining
once at the end is typically both clearer and no worse, and scales better
as the piece count grows:

```swift
measure("string += in loop") {
    var s = ""
    for i in 0..<20_000 { s += "\(i)" }
    _ = s.count
}

measure("array join") {
    var parts: [String] = []
    parts.reserveCapacity(20_000)
    for i in 0..<20_000 { parts.append("\(i)") }
    let s = parts.joined()
    _ = s.count
}
```

## Trap 3: `Array.contains` where a `Set` belongs

`Array.contains` is O(n) — it scans element by element. `Set.contains` is
O(1) average, backed by a hash table. For any "is this value present"
check done repeatedly, converting to a `Set` up front pays for itself
almost immediately:

```swift
let numbers = Array(0..<50_000)
let numberSet = Set(numbers)

measure("Array.contains (O(n))") {
    for target in [1, 100, 49_999] {
        _ = numbers.contains(target)
    }
}

measure("Set.contains (O(1))") {
    for target in [1, 100, 49_999] {
        _ = numberSet.contains(target)
    }
}
```

## Full run and real output

Running the whole file unoptimized (`swift performance.swift` — no `-O`,
so the gaps are visible rather than erased by the optimizer):

```
append without reserve: 30.44ms
append with reserveCapacity: 27.05ms
string += in loop: 3.76ms
array join: 4.00ms
Array.contains (O(n)): 3.94ms
Set.contains (O(1)): 0.00ms
Before mutation, arrays share storage (COW): equal? true
After mutating copy: original = [1, 2, 3] copy = [1, 2, 3, 4]
```

`Set.contains` rounding to `0.00ms` against `Array.contains`'s ~4ms for
just three lookups is the clearest signal here — the gap only widens as
either the collection size or the number of lookups grows. The
`reserveCapacity` gap is real but modest at this size; it becomes far more
pronounced on arrays with millions of elements or expensive-to-copy
elements. **Always measure with `-O`** (`swift -O file.swift`, or a Release
build) before drawing conclusions for production code — the debug numbers
above exaggerate differences the optimizer often narrows or removes
entirely; the exercise is to build the intuition for *which* patterns to
suspect, not to memorize these exact numbers.

## Copy-on-write: why arrays are cheap to pass around

`Array`, `Dictionary`, `Set`, and `String` are all copy-on-write (COW):
assigning one to another doesn't copy the underlying buffer, only a
reference to it — the actual copy happens lazily, only when one of the two
is mutated:

```swift
var original = [1, 2, 3]
var copy = original
print("Before mutation, arrays share storage (COW): equal?", original == copy)
copy.append(4)
print("After mutating copy: original =", original, "copy =", copy)
```

Output:

```
Before mutation, arrays share storage (COW): equal? true
After mutating copy: original = [1, 2, 3] copy = [1, 2, 3, 4]
```

This is why passing an array into a function is cheap even for a huge
array — no copy happens unless the function actually mutates its own
(value-semantic) parameter.

## Real profiling: Instruments

Hand-rolled timers are useful for A/B-ing a specific hypothesis, but for
finding *unknown* hot spots in a real app, Xcode's Instruments is the right
tool — the **Time Profiler** template samples the call stack repeatedly
and shows which functions consume the most wall-clock time, and the
**Allocations** template tracks memory growth over a run to catch leaks and
unexpected retention. Both require the full Xcode app (not available in
this CLT-only environment) to launch and attach to a running process.

## Swift-specific traps

- **Debug builds (`swift file.swift`, or an Xcode Debug scheme) can be an
  order of magnitude slower than `-O`/Release** — never conclude "Swift is
  slow at X" from an unoptimized run; always compare optimized numbers
  before deciding something needs to change.
- **`reserveCapacity` only helps if you know the size ahead of time** —
  calling it with a wrong (too-small) estimate provides no benefit, since
  the array still grows dynamically past that point.
- **`Set` and `Dictionary` require `Hashable` elements/keys** — a custom
  struct needs a correct `Hashable` conformance (usually synthesized for
  free when every stored property is itself `Hashable`) before it can go in
  either.
- **COW doesn't make mutation free** — the *first* mutation after a copy
  still pays the full copy cost; COW only avoids the copy for arrays that
  are copied but never mutated.

## Cheat sheet

| Symptom | Likely fix |
|---------|------------|
| Slow loop appending to an array of known final size | `array.reserveCapacity(n)` |
| Slow loop building a big string | Collect pieces in an array, `joined()` once |
| Repeated `.contains(...)` checks on a large collection | Convert to `Set` once, reuse it |
| Need to find an unknown hot spot | Instruments' Time Profiler (needs full Xcode) |
| Suspected memory leak/growth | Instruments' Allocations template (needs full Xcode) |

## Exercise

Write two versions of a function that deduplicates a `[Int]` while
preserving order: one using `Array.contains` inside the loop (O(n²)
overall) and one using a `Set` to track seen values (O(n) overall). Time
both with the `measure` helper above on an array of at least 50,000
elements with many duplicates, and confirm the `Set`-based version is
faster. Report both timings in a comment.
