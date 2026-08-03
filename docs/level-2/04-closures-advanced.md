# 04 · Closures Advanced

[Level 1's closure lesson](../level-1/09-closures-intro.md) covered syntax
and basic capturing. Now the harder question: **what exactly does a closure
capture, and how long does it live?** Get this wrong and you get retain
cycles (memory leaks) or closures that unexpectedly hold onto stale data —
two of the most common intermediate-level Swift bugs.

## Capture by reference, not by value

By default, a closure captures variables **by reference** — it doesn't
freeze the value at closure-creation time, it keeps a live link to the
variable itself:

```swift
var counter = 0

let incrementAndPrint = {
    counter += 1
    print("Counter is now \(counter)")
}

incrementAndPrint()   // Counter is now 1
counter = 100          // mutated from OUTSIDE the closure
incrementAndPrint()   // Counter is now 101 -- sees the external change
```

If closures captured by value, the second call would print `2`, not `101`.
This "live reference" behavior is exactly what powers patterns like
`makeIncrementer` from Level 1.

## Capture lists: freezing a value at creation time

Sometimes you *want* the value as it was when the closure was created, not a
live reference. A **capture list** — square brackets right after `{` —
does that:

```swift
var score = 10

let liveClosure = {
    print("Live: \(score)")
}

let frozenClosure = { [score] in
    print("Frozen: \(score)")   // "score" here is a new constant, captured at creation
}

score = 999

liveClosure()     // Live: 999
frozenClosure()   // Frozen: 10
```

`[score]` creates a new constant named `score` inside the closure, initialized
to the *current* value at the moment the closure is created — later changes
to the outer `score` don't affect it. This is invaluable for capturing
loop variables or state you want "snapshotted."

## `@escaping` vs non-escaping closures

A closure parameter is **non-escaping** by default: it's guaranteed to be
called before the function returns, so Swift lets the compiler make
optimizations and doesn't require special handling of `self`. An
**escaping** closure is stored somewhere and called *after* the function has
already returned — for example, a completion handler kept around for a
network callback:

```swift
// Non-escaping (default): the closure is used and done before the function returns
func performTwice(_ action: () -> Void) {
    action()
    action()
}

// Escaping: the closure outlives the function call
var savedCompletions: [() -> Void] = []

func performLater(_ action: @escaping () -> Void) {
    savedCompletions.append(action)   // stored for later -- must be @escaping
}

performLater { print("Called later!") }
print("performLater returned, nothing printed yet")
savedCompletions.forEach { $0() }
// performLater returned, nothing printed yet
// Called later!
```

Without `@escaping`, the line `savedCompletions.append(action)` wouldn't
compile — the compiler can prove the closure needs to survive past the
function's return, and it forces you to acknowledge that explicitly.

## The retain cycle trap

Classes are reference types managed by ARC (Automatic Reference Counting):
an instance is deallocated once nothing holds a **strong** reference to it.
An escaping closure that captures `self` strongly, while `self` also holds
onto that closure, creates a cycle — neither can ever reach zero references,
so neither is ever freed:

```swift
class DataLoader {
    var onComplete: (() -> Void)?
    let name: String

    init(name: String) {
        self.name = name
    }

    func startLoading() {
        // BUG: this closure captures "self" strongly, and self.onComplete
        // holds the closure -- a retain cycle. DataLoader can never deinit.
        onComplete = {
            print("\(self.name) finished loading")
        }
    }

    deinit {
        print("\(name) deinitialized")
    }
}

// var loader: DataLoader? = DataLoader(name: "Loader A")
// loader?.startLoading()
// loader = nil
// -- "Loader A deinitialized" never prints: the closure keeps it alive forever
```

## Breaking the cycle with `[weak self]`

Adding `self` to the capture list as `weak` breaks the strong reference —
the closure holds a weak (optional) link that becomes `nil` automatically if
the object is deallocated elsewhere:

```swift
class FixedDataLoader {
    var onComplete: (() -> Void)?
    let name: String

    init(name: String) {
        self.name = name
    }

    func startLoading() {
        onComplete = { [weak self] in
            // self is now Self? -- must be unwrapped
            guard let self = self else {
                print("Loader was already deallocated")
                return
            }
            print("\(self.name) finished loading")
        }
    }

    deinit {
        print("\(name) deinitialized")
    }
}

var loader: FixedDataLoader? = FixedDataLoader(name: "Loader B")
loader?.startLoading()
loader?.onComplete?()
loader = nil
// Loader B finished loading
// Loader B deinitialized
```

`guard let self = self else { return }` is the standard pattern: it
temporarily creates a strong reference for the duration of the closure body
(so `self` doesn't vanish mid-execution) without holding one permanently
between calls.

## `[unowned self]`: the sharper alternative

`unowned` also avoids the retain cycle, but — unlike `weak` — it's not
optional, and it **crashes** if you access it after the object is gone. Use
it only when you're certain the closure can never outlive `self`:

```swift
class Repeater {
    let name: String
    init(name: String) { self.name = name }

    lazy var greet: () -> Void = { [unowned self] in
        print("Hi from \(self.name)")   // no unwrapping needed, but unsafe if self is gone
    }
}

let repeater = Repeater(name: "R2")
repeater.greet()   // Hi from R2
```

`[weak self]` is the safer default; reach for `[unowned self]` only for
tightly coupled closures (like a lazy property referencing its own instance)
where you can prove the lifetimes are locked together.

## Cheat sheet

| Capture style | Syntax | Behavior |
|----------------|--------|----------|
| Default (by reference) | `{ ... }` | Live link to the variable; sees later mutations |
| Capture list, by value | `{ [x] in ... }` | Snapshots `x`'s value at closure creation |
| Weak capture | `{ [weak self] in ... }` | Optional reference; `nil` if the object is gone; safest for escaping closures |
| Unowned capture | `{ [unowned self] in ... }` | Non-optional reference; crashes if accessed after deallocation |
| Non-escaping param (default) | `(() -> Void)` | Must be called before the function returns |
| Escaping param | `(@escaping () -> Void)` | Can be stored and called after the function returns |

## Exercise

Build a small `class Timer` with a stored `onFire: (() -> Void)?` and a
method `simulateFire()` that just calls `onFire?()`. Create a second class,
`class Widget`, with a `name: String` and a method `startTimer(_ timer:
Timer)` that sets `timer.onFire` to a closure printing `"\(name) fired"`. Do
it first with a plain (strongly capturing) closure and add a `deinit` to
`Widget` that prints when it's deallocated — set the widget to `nil` and
observe that `deinit` never runs. Then fix it with `[weak self]` and confirm
`deinit` now prints as expected.
