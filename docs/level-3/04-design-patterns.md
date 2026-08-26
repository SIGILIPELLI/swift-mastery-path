# 04 · Design Patterns in Swift

Classic object-oriented design patterns still apply in Swift, but Swift's
value types, protocols, and closures often give you a lighter-weight way to
express the same idea than the class-hierarchy version from a Java or C++
textbook. This module walks through four patterns you'll actually reach for.

## Singleton

A singleton guarantees exactly one shared instance, reached through a
static property. `static let` is thread-safe by construction — Swift
initializes it lazily, exactly once, even under concurrent access:

```swift
final class Logger {
    static let shared = Logger()
    private init() {}   // prevents `Logger()` from outside
    private(set) var messages: [String] = []
    func log(_ message: String) { messages.append(message) }
}

Logger.shared.log("App started")
Logger.shared.log("User signed in")
print("Logged messages:", Logger.shared.messages)
```

Singletons are convenient but make testing harder (shared global state
persists between tests) — prefer passing a `Logger` instance explicitly
where you can, and reserve real singletons for things that are genuinely
unique per process, like a shared `URLSession` or a log destination.

## Factory

A factory centralizes the logic for choosing *which* concrete type to
create, so callers depend only on a protocol:

```swift
protocol Shape { func area() -> Double }
struct Circle: Shape { let radius: Double; func area() -> Double { .pi * radius * radius } }
struct Square: Shape { let side: Double; func area() -> Double { side * side } }

enum ShapeKind { case circle(radius: Double), square(side: Double) }

enum ShapeFactory {
    static func make(_ kind: ShapeKind) -> Shape {
        switch kind {
        case .circle(let radius): return Circle(radius: radius)
        case .square(let side): return Square(side: side)
        }
    }
}

let shapes = [ShapeFactory.make(.circle(radius: 2)), ShapeFactory.make(.square(side: 3))]
for shape in shapes {
    print(String(format: "Area: %.2f", shape.area()))
}
```

Using an `enum` as the factory's input (rather than exposing `Circle` and
`Square` directly) keeps construction details out of calling code — adding
a new shape case means changing the factory in one place.

## Observer

The Observer pattern lets one object broadcast changes to any number of
interested listeners without knowing who they are. Swift's `didSet`
property observer is a natural hook for this:

```swift
protocol PriceObserver: AnyObject { func priceDidChange(to price: Double) }

final class Stock {
    private var observers: [PriceObserver] = []
    var price: Double = 0 {
        didSet { observers.forEach { $0.priceDidChange(to: price) } }
    }
    func subscribe(_ observer: PriceObserver) { observers.append(observer) }
}

final class PriceLogger: PriceObserver {
    func priceDidChange(to price: Double) { print("Price changed to \(price)") }
}

let stock = Stock()
let priceLogger = PriceLogger()
stock.subscribe(priceLogger)
stock.price = 101.5
stock.price = 99.0
```

`PriceObserver: AnyObject` constrains the protocol to classes, which is
necessary here because `Stock` stores observers by reference and needs
identity semantics (to eventually support `unsubscribe`, for example) —
Combine's `@Published` and SwiftUI's observation machinery are built on the
same underlying idea.

## Strategy

Strategy swaps out an algorithm at runtime behind a shared interface. In
Swift, a protocol (or often just a closure) is the interface:

```swift
protocol DiscountStrategy { func apply(to total: Double) -> Double }
struct NoDiscount: DiscountStrategy { func apply(to total: Double) -> Double { total } }
struct PercentOff: DiscountStrategy {
    let percent: Double
    func apply(to total: Double) -> Double { total * (1 - percent / 100) }
}

struct Cart {
    var total: Double
    var strategy: DiscountStrategy
    func finalPrice() -> Double { strategy.apply(to: total) }
}

let cart = Cart(total: 200, strategy: PercentOff(percent: 10))
print("Final price:", cart.finalPrice())
```

## Full output

Running all four sections together (`swift design-patterns.swift`):

```
Logged messages: ["App started", "User signed in"]
Area: 12.57
Area: 9.00
Price changed to 101.5
Price changed to 99.0
Final price: 180.0
```

## Swift-specific traps

- **`static let` is the idiomatic singleton — don't hand-roll one with
  `dispatch_once`-style code** (a pattern you may see in old
  Objective-C-derived tutorials); Swift's static stored properties are
  already lazy and thread-safe.
- **Observer protocols usually need `AnyObject`.** Forgetting the class
  constraint means you can't store observers in a way that supports
  identity comparison or weak references, and Swift will let you write code
  that silently retains observers forever (a memory leak) if you're not
  careful — consider wrapping observer references as `weak` in a real
  implementation.
- **A `struct`-based Strategy loses the "reconfigure later" flexibility a
  `class` gives you for free**, but that's often a feature: `Cart` above is
  copied by value, so two carts never accidentally share a discount
  strategy instance's mutable state.
- **Factories that return a protocol type erase the concrete type** — code
  calling `ShapeFactory.make` cannot later do `if let circle = shape as?
  Circle` without an explicit downcast, which is usually the point, but can
  surprise people expecting the concrete type back.

## Cheat sheet

| Pattern | Swift idiom |
|---------|-------------|
| Singleton | `static let shared = Type()` + `private init()` |
| Factory | An `enum` or function returning a protocol type |
| Observer | A protocol (`AnyObject`-constrained) + a list of subscribers |
| Strategy | A protocol (or closure) stored as a property and swapped at runtime |

## Exercise

Implement the Decorator pattern: define a `protocol Coffee { func cost() ->
Double; func description() -> String }`, a base `struct Espresso: Coffee`
returning `1.50` and `"Espresso"`, and two decorator structs — `MilkAdded`
and `SyrupAdded` — that each wrap an existing `Coffee` value, add to its
cost, and append to its description. Compose `SyrupAdded(MilkAdded(Espresso()))`
and print both its final cost and full description.
