# 07 · Structs & Classes Basics

Swift gives you two ways to bundle data and behavior: `struct` (a **value
type**) and `class` (a **reference type**). Apple's own guidance, and Swift's
standard library, favor structs by default — reach for a class only when you
specifically need reference semantics or inheritance.

## Defining a struct

```swift
struct Point {
    var x: Double
    var y: Double

    // computed property
    var distanceFromOrigin: Double {
        (x * x + y * y).squareRoot()
    }

    func offsetBy(dx: Double, dy: Double) -> Point {
        Point(x: x + dx, y: y + dy)
    }
}

let origin = Point(x: 0, y: 0)          // memberwise initializer, free
let p1 = Point(x: 3, y: 4)
print(p1.distanceFromOrigin)             // 5.0

let p2 = p1.offsetBy(dx: 1, dy: 1)
print(p2.x, p2.y)                        // 4.0 5.0
```

Every struct gets a **memberwise initializer** for free — `Point(x:y:)` above
was never written explicitly.

## Value semantics — the key difference

```swift
struct Counter {
    var count = 0
}

var original = Counter()
var copy = original         // COPIES the value
copy.count = 100

print(original.count)   // 0 -- unaffected by the copy's mutation
print(copy.count)        // 100
```

Compare the same scenario with a class:

```swift
class CounterBox {
    var count = 0
}

let original = CounterBox()
let copy = original          // COPIES the reference -- both point to the same object
copy.count = 100

print(original.count)   // 100 -- same underlying object!
print(copy.count)        // 100
```

This is the single most important thing to internalize about Swift: structs
copy on assignment (independent values), classes share on assignment
(same underlying instance via a reference).

## Mutating methods

Struct methods that modify `self` must be marked `mutating` (the struct
instance itself must also be a `var`, not `let`):

```swift
struct Counter {
    var count = 0

    mutating func increment() {
        count += 1
    }
}

var counter = Counter()
counter.increment()
counter.increment()
print(counter.count)   // 2

let frozenCounter = Counter()
// frozenCounter.increment()   // compile error: cannot mutate a "let" struct
```

## Defining a class

```swift
class Vehicle {
    var speed: Double = 0
    let make: String

    init(make: String) {
        self.make = make
    }

    func accelerate(by amount: Double) {
        speed += amount
    }

    func describe() -> String {
        "\(make) traveling at \(speed) mph"
    }
}

let car = Vehicle(make: "Toyota")
car.accelerate(by: 30)
print(car.describe())   // Toyota traveling at 30.0 mph
```

Unlike structs, classes require an explicit `init` once you declare any
non-defaulted stored property — there's no free memberwise initializer.

## Inheritance (classes only)

```swift
class ElectricVehicle: Vehicle {
    var batteryPercent: Double = 100

    override func describe() -> String {
        super.describe() + ", battery at \(batteryPercent)%"
    }
}

let tesla = ElectricVehicle(make: "Tesla")
tesla.accelerate(by: 60)
print(tesla.describe())
// Tesla traveling at 60.0 mph, battery at 100.0%
```

Structs cannot inherit from another struct — for shared behavior across
structs, Swift uses protocols and extensions instead (covered in
[Level 2](../level-2/01-oop-deep-dive.md)).

## `struct` vs `class`

| Aspect | `struct` | `class` |
|--------|----------|---------|
| Semantics | Value (copied on assignment) | Reference (shared on assignment) |
| Inheritance | No | Yes (single inheritance) |
| Free initializer | Memberwise, automatic | No — must write `init` yourself |
| Mutation | Needs `mutating` methods, instance must be `var` | Any method can mutate properties freely |
| Deinitializers | No | Yes (`deinit`) |
| Typical use | Most data models: points, records, configs | Shared, identity-based objects; UI controllers |
| Default choice | **Yes, start here** | Only when you need reference semantics or inheritance |

## Exercise

Define a `struct BankAccount` with a `var balance: Double`, and `mutating`
methods `deposit(_:)` and `withdraw(_:)` (withdraw should refuse — print an
error and not mutate `balance` — if the amount exceeds the balance). Create
two `var` accounts, copy one into the other, mutate the copy, and print both
balances to confirm they're independent. Then define a `class Logger` with
an array of message strings and an `append(_:)` method; create one instance,
assign it to a second variable, mutate through the second, and print the
first to confirm they share state.
