# 01 · OOP Deep Dive

Level 1 introduced structs and classes as two ways to bundle data and
behavior. This module goes deeper into how Swift actually does
object-oriented programming — and the twist is that idiomatic Swift leans
less on classical inheritance than languages like Java or C++, and more on
**protocols** and **extensions**. Understanding when to reach for which tool
is the core skill this lesson builds.

## Classes vs structs, revisited

The Level 1 table covered value vs reference semantics. Here's the practical
consequence for API design — a shared, mutable class instance can be handed
to multiple owners who all see each other's changes, which is sometimes
exactly what you want:

```swift
class BankAccount {
    private(set) var balance: Double

    init(balance: Double) {
        self.balance = balance
    }

    func deposit(_ amount: Double) {
        balance += amount
    }
}

let account = BankAccount(balance: 100)
let sameAccount = account          // reference copy -- same object
sameAccount.deposit(50)
print(account.balance)             // 150.0 -- both variables see the deposit
```

`private(set)` lets anyone read `balance` but only code inside `BankAccount`
can change it — useful for protecting invariants like "balance never goes
negative directly." If `BankAccount` were a struct, `sameAccount` would be an
independent copy, and the deposit would silently only affect the copy — a
common source of confusing bugs when reference semantics are actually
needed (shared caches, view controllers, connection pools).

## Inheritance and `override`

Classes support single inheritance. A subclass can override methods and
computed properties, and must call `super` to get the base class's behavior
if it still wants it:

```swift
class Shape {
    var name: String
    init(name: String) { self.name = name }

    func area() -> Double { 0 }

    func describe() -> String {
        "\(name): area = \(area())"
    }
}

class Circle: Shape {
    var radius: Double

    init(radius: Double) {
        self.radius = radius
        super.init(name: "Circle")   // base class init must run before use
    }

    override func area() -> Double {
        Double.pi * radius * radius
    }
}

let circle = Circle(radius: 2)
print(circle.describe())   // Circle: area = 12.566370614359172
```

Swift requires `override` explicitly (unlike some languages where it's
optional) — this catches typos where you meant to override a method but
misspelled its name, which would otherwise silently create a brand-new
method instead of an error.

## Protocols: Swift's real workhorse

A protocol defines a contract — a set of methods and properties a type
promises to implement — without dictating *how*. Any type (struct, class, or
enum) can conform:

```swift
protocol Describable {
    var name: String { get }
    func describe() -> String
}

struct Product: Describable {
    let name: String
    let price: Double

    func describe() -> String {
        "\(name) costs $\(price)"
    }
}

enum Status: Describable {
    case active, inactive

    var name: String {
        switch self {
        case .active: return "Active"
        case .inactive: return "Inactive"
        }
    }

    func describe() -> String {
        "Status is \(name)"
    }
}

let items: [Describable] = [
    Product(name: "Keyboard", price: 49.99),
    Status.active
]
for item in items {
    print(item.describe())
}
// Keyboard costs $49.99
// Status is Active
```

This is the payoff: a struct and an enum — types that can never share a
common superclass — both satisfy `Describable`, and both can live in the
same `[Describable]` array. Protocols give you polymorphism across
completely unrelated types, something class inheritance alone can't do.

## Extensions: adding behavior after the fact

An extension adds methods, computed properties, or protocol conformances to
an *existing* type — even ones you don't own, like `Int` or `String`:

```swift
extension Int {
    var isEven: Bool { self % 2 == 0 }

    func squared() -> Int { self * self }
}

print(4.isEven)      // true
print(5.squared())   // 25
```

Extensions are also how you split a type's protocol conformance into a
separate, clearly labeled block — a common Swift style for readability:

```swift
struct Point {
    var x: Double
    var y: Double
}

extension Point: Describable {
    var name: String { "Point(\(x), \(y))" }
    func describe() -> String { "\(name), distance from origin: \((x * x + y * y).squareRoot())" }
}

print(Point(x: 3, y: 4).describe())
// Point(3.0, 4.0), distance from origin: 5.0
```

## Protocol extensions: default implementations

You can extend a *protocol itself* to give every conforming type a default
implementation for free — conforming types only need to override it if they
want different behavior:

```swift
protocol Greetable {
    var name: String { get }
    func greet() -> String
}

extension Greetable {
    // Default implementation -- any conforming type gets this automatically
    func greet() -> String {
        "Hello, \(name)!"
    }
}

struct Person: Greetable {
    let name: String
    // No greet() needed -- uses the protocol extension's default
}

struct Robot: Greetable {
    let name: String
    func greet() -> String {   // opts out of the default
        "BEEP BOOP \(name.uppercased())"
    }
}

print(Person(name: "Ada").greet())    // Hello, Ada!
print(Robot(name: "T-800").greet())   // BEEP BOOP T-800
```

This pattern — protocol + protocol extension — is often called **protocol-
oriented programming**, and Apple explicitly recommends it over deep class
hierarchies. It gives you shared behavior (like inheritance) plus the
flexibility to mix it into any type (like composition), without the fragile
base-class problems that come with multi-level inheritance.

## Composing behavior with multiple protocols

A type can conform to several protocols at once, composing capabilities:

```swift
protocol Flyable {
    func fly() -> String
}

protocol Swimmable {
    func swim() -> String
}

struct Duck: Flyable, Swimmable {
    func fly() -> String { "Flying low over the pond" }
    func swim() -> String { "Paddling across the pond" }
}

func describeAbilities(_ animal: Flyable & Swimmable) {
    print(animal.fly())
    print(animal.swim())
}

describeAbilities(Duck())
```

`Flyable & Swimmable` is a **protocol composition type** — it accepts only
values that conform to *both*, which is far more flexible than requiring a
specific base class that happens to implement both abilities.

## `final`: opting out of further overriding

Marking a class or method `final` prevents subclassing or overriding —
useful for performance (the compiler can skip dynamic dispatch) and for
signaling "this behavior is not meant to change":

```swift
final class Configuration {
    let apiKey: String
    init(apiKey: String) { self.apiKey = apiKey }
}

// class DebugConfiguration: Configuration {}   // compile error: cannot inherit from final class
```

## Cheat sheet

| Tool | Use it when |
|------|-------------|
| `class` inheritance | You need shared, mutable state and a genuine "is-a" relationship, usually 1-2 levels deep |
| `protocol` | You want unrelated types (struct, class, enum) to share a contract |
| `protocol extension` | You want a default implementation that conforming types can override |
| `extension` on a concrete type | You want to add behavior to a type without touching its original definition |
| `Protocol & Protocol` | A function needs to accept "anything that can do both X and Y" |
| `final` | You want to lock a class against subclassing, or a method against overriding |

## Exercise

Define a protocol `Payable` with a computed property `amountDue: Double` and
a method `pay() -> String`. Add a protocol extension giving `pay()` a
default implementation that returns `"Paid $\(amountDue)"`. Then create a
`struct Invoice: Payable` and a `class Subscription: Payable` (each storing
whatever fields make sense), put instances of both in a `[Payable]` array,
and print `pay()` for each. Finally, override `pay()` in `Subscription` to
return a custom message instead of the default, and confirm the array loop
now prints the override only for subscriptions.
