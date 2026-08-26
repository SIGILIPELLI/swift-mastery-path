# 09 · Protocol-Oriented Programming

Swift is often described as protocol-oriented rather than purely
object-oriented: protocols with default implementations (via extensions)
let you share behavior across unrelated types — structs, enums, and classes
alike — without inheritance. This module builds up the toolkit: default
implementations, associated types, composition, and retroactive
conformance.

## Default implementations via protocol extensions

A protocol extension can supply a default body for a requirement; any
conforming type gets it for free unless it defines its own:

```swift
protocol Describable {
    var description: String { get }
}

extension Describable {
    var description: String { "A \(type(of: self))" }   // default implementation
}

struct Widget: Describable {}
struct Gadget: Describable {
    var description: String { "A custom Gadget" }   // overrides the default
}

print(Widget().description)
print(Gadget().description)
```

Output:

```
A Widget
A custom Gadget
```

`Widget` gets the default `description` untouched; `Gadget` supplies its
own, which Swift dispatches to statically (via the type's own conformance)
rather than through the extension's default.

## Associated types: protocols with a generic "hole"

`associatedtype` lets a protocol describe a requirement whose concrete type
is decided by each conforming type — this is how `Container`-style
protocols work without knowing in advance what they'll hold:

```swift
protocol Container {
    associatedtype Item
    var items: [Item] { get set }
    mutating func add(_ item: Item)
    var count: Int { get }
}

extension Container {
    var count: Int { items.count }   // one default, shared by every conformer
}

struct Stack<Element>: Container {
    var items: [Element] = []
    mutating func add(_ item: Element) { items.append(item) }
    mutating func pop() -> Element? { items.popLast() }
}

var intStack = Stack<Int>()
intStack.add(1)
intStack.add(2)
intStack.add(3)
print("Stack count:", intStack.count)
print("Popped:", intStack.pop() ?? -1)
```

Output:

```
Stack count: 3
Popped: 3
```

`Stack<Element>` satisfies `Container`'s `associatedtype Item` by inferring
`Item == Element` from its own generic parameter and the `items: [Element]`
property — no explicit `typealias Item = Element` is needed here because
the compiler can infer it.

## Protocol composition

`&` combines multiple protocols into a single requirement a function can
accept, without needing a named type that inherits from both:

```swift
protocol Named { var name: String { get } }
protocol Aged { var age: Int { get } }

struct Person: Named, Aged {
    let name: String
    let age: Int
}

func greet(_ subject: Named & Aged) {
    print("\(subject.name) is \(subject.age) years old")
}
greet(Person(name: "Ada", age: 30))
```

Output:

```
Ada is 30 years old
```

## Constrained extensions with `where`

You can extend a protocol *only* for conformers meeting an extra condition
— here, only collections whose `Element` is `Numeric`:

```swift
extension Collection where Element: Numeric {
    func sum() -> Element {
        reduce(0, +)
    }
}
print("Sum:", [1, 2, 3, 4].sum())
```

Output:

```
Sum: 10
```

`[String]` doesn't get a `.sum()` method at all — the `where Element:
Numeric` clause means this extension's method simply doesn't exist for
non-numeric collections, which is enforced at compile time, not by a
runtime check.

## Retroactively conforming an existing type

Any type — including ones from the standard library — can be made to
conform to a protocol you define later, as long as it doesn't clash with an
existing member of the same name:

```swift
protocol Greetable {
    func greeting() -> String
}
extension Greetable {
    func greeting() -> String { "Hello from \(type(of: self))" }
}
extension Int: Greetable {}
print(42.greeting())
```

Output:

```
Hello from Int
```

## Full run

All of the above, run together (`swift protocol-oriented.swift`):

```
A Widget
A custom Gadget
Stack count: 3
Popped: 3
Ada is 30 years old
Sum: 10
Hello from Int
```

## Swift-specific traps

- **Default implementations are statically dispatched when accessed
  through a concrete type, but dynamically dispatched when accessed
  through the protocol type** — if you assign a `Gadget` to a `let x:
  Describable` and both `Gadget` and the extension define `description`,
  `x.description` still correctly calls `Gadget`'s own implementation
  (real protocol requirements dispatch dynamically); it's *non-requirement*
  extension methods that only exist in the extension where static/dynamic
  dispatch can surprise you.
- **Retroactive conformance can silently conflict** with an existing member
  of the same name and signature — as seen with `Int` and `description`
  above (already claimed by `CustomStringConvertible`/`BinaryInteger`) —
  pick names unlikely to collide, or check first.
- **`associatedtype` requirements make a protocol usable only as a generic
  constraint (`some Container`, `<T: Container>`), not as a plain existential
  type (`any Container`) for most uses** without Swift 5.7+'s
  `primary associated types` / existential improvements — a common
  "protocol can only be used as a generic constraint" compiler error.
- **A `where` clause on a protocol extension doesn't restrict conformance**
  — it restricts which conformers get *that particular extension's*
  members; `[String]` is still a fine `Collection`, it just doesn't gain
  `.sum()`.

## Cheat sheet

| Feature | Purpose |
|---------|---------|
| `extension P { ... }` | Default implementation for a protocol requirement |
| `associatedtype Item` | A protocol-level generic placeholder |
| `A & B` | Require conformance to multiple protocols at once |
| `extension P where Self: X` / `where Element: Y` | Conditional extension members |
| `extension ExistingType: NewProtocol {}` | Retroactive conformance |

## Exercise

Define a protocol `Flyable` with a default-implemented `func fly() ->
String` returning `"Flying"`, and a protocol `Swimmable` with a
default-implemented `func swim() -> String` returning `"Swimming"`. Create
a `struct Duck` conforming to both, plus a function `func describe(_ x:
Flyable & Swimmable)` that prints both capabilities. Then give `Duck` its
own override of `fly()` returning `"Duck flying low over the pond"` and
confirm `describe` picks up the override while `swim()` still uses the
shared default.
