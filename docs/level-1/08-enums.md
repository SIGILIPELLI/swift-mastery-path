---
description: "Enums — Swift enums are far more capable than a simple list of named constants — they can carry associated data per case, have methods, conform to…"
---

# 08 · Enums

Swift enums are far more capable than a simple list of named constants —
they can carry associated data per case, have methods, conform to
protocols, and are a first-class tool for modeling state.

## Basic enums

```swift
enum Direction {
    case north
    case south
    case east
    case west
}

let heading = Direction.north

// Type can be inferred at the use site once the variable's type is known:
var currentDirection: Direction = .south

switch currentDirection {
case .north:
    print("Heading up")
case .south:
    print("Heading down")
case .east, .west:
    print("Heading sideways")
}
// Heading down
```

`switch` over an enum must be exhaustive — add a new case later and the
compiler flags every `switch` that forgot to handle it, a huge help when
refactoring.

## Raw values

```swift
enum Direction: String {
    case north = "N"
    case south = "S"
    case east = "E"
    case west = "W"
}

print(Direction.north.rawValue)   // N

let fromRaw = Direction(rawValue: "S")
print(fromRaw!)                    // south
print(Direction(rawValue: "Q"))     // nil -- invalid raw value
```

```swift
enum Priority: Int {
    case low = 1
    case medium
    case high
}
print(Priority.medium.rawValue)   // 2 -- auto-incremented from "low"
```

## Associated values

Unlike raw values (one fixed value per case, shared across all instances of
that case), associated values let each *instance* of a case carry its own
data.

```swift
enum NetworkResult {
    case success(data: String)
    case failure(code: Int, message: String)
    case loading
}

func handle(_ result: NetworkResult) {
    switch result {
    case .success(let data):
        print("Got data: \(data)")
    case .failure(let code, let message):
        print("Error \(code): \(message)")
    case .loading:
        print("Still loading...")
    }
}

handle(.success(data: "user profile"))
// Got data: user profile
handle(.failure(code: 404, message: "Not Found"))
// Error 404: Not Found
handle(.loading)
// Still loading...
```

This is the same shape as `Optional` itself, which is really just an enum
under the hood:

```swift
// Swift's standard library, conceptually:
// enum Optional<Wrapped> {
//     case none
//     case some(Wrapped)
// }
```

## Methods and computed properties on enums

```swift
enum Direction: String {
    case north = "N", south = "S", east = "E", west = "W"

    func opposite() -> Direction {
        switch self {
        case .north: return .south
        case .south: return .north
        case .east: return .west
        case .west: return .east
        }
    }
}

print(Direction.north.opposite())   // south
```

## `CaseIterable`

```swift
enum Suit: CaseIterable {
    case clubs, diamonds, hearts, spades
}

for suit in Suit.allCases {
    print(suit)
}
// clubs
// diamonds
// hearts
// spades

print(Suit.allCases.count)   // 4
```

## Enums with methods that mutate `self`

```swift
enum TrafficLight {
    case red, yellow, green

    mutating func next() {
        switch self {
        case .red: self = .green
        case .green: self = .yellow
        case .yellow: self = .red
        }
    }
}

var light = TrafficLight.red
light.next()
print(light)   // green
light.next()
print(light)   // yellow
```

## Cheat sheet

| Feature | Example | Notes |
|---------|---------|-------|
| Basic case | `case north` | No associated data |
| Raw value | `case low = 1` | One fixed value per case, shared by all instances |
| Associated value | `case success(data: String)` | Different data per instance of the case |
| `rawValue` init | `Direction(rawValue: "N")` | Returns an `Optional` — invalid input yields `nil` |
| `CaseIterable` | `Suit.allCases` | Iterate every case automatically |
| `mutating func` | Changes `self` to a different case | Requires the enum variable to be `var` |

## How It Actually Works

**An enum without associated values is just a small integer under the
hood.** With ≤256 cases and no payload, Swift represents a value as a
single byte holding a discriminator tag — `Direction.north` compiles to
essentially the constant `0`. The compiler picks the smallest integer width
that fits the case count, so simple enums are as cheap as a raw `Int8`, and
`switch` over one compiles to a jump table, not a chain of comparisons —
that's part of why exhaustiveness matters: the compiler is building a table
sized to the case count, and a new case genuinely requires a new table
entry, which is what it flags as a compile error.

**Enums with associated values are tagged unions** — the same
discriminator tag, plus enough storage for the *largest* case's payload.
`NetworkResult`'s layout reserves space for a tag byte and the largest of
`(String)`, `(Int, String)`, or nothing — every instance is sized for the
worst case, with unused bytes for smaller cases. This is exactly what a C
`union` with a manual tag field does, except the compiler generates and
enforces the tag/switch correspondence for you, which is why you can't read
`.success`'s `data` without going through a `switch` or `if case` — there's
no type-safe way to know which payload variant is live except by checking
the tag.

**Indirect cases and recursion**: an enum case whose payload contains the
enum itself (e.g. a `case node(value: Int, next: LinkedList)`) can't have a
fixed size — the compiler would need infinite storage to lay it out
directly. Marking the case (or the whole enum) `indirect` makes Swift box
that case's payload on the heap and store just a pointer inline, exactly
the same mechanism a class provides implicitly, opted into per-case.

**Why `Optional` really is "just an enum"**: `Optional<Wrapped>.none` and
`.some(Wrapped)` follow the identical tagged-union layout described above —
which is also why `if let` and `switch` work uniformly across `Optional`
and your own enums; they're the same runtime shape, not special-cased
syntax.

## 🔀 See this in another language

- [Kotlin — Data Classes](https://sigilipelli.github.io/kotlin-mastery-path/level-1/08-data-classes/)
- [Shell/Bash — Basic Text Processing](https://sigilipelli.github.io/shell-mastery-path/level-1/08-text-processing/)
- [C — File I/O](https://sigilipelli.github.io/c-mastery-path/level-1/08-file-io/)

## Exercise

Model a vending machine with `enum VendingItem: String, CaseIterable { case soda, chips, candy }`
where each case has a price via a computed property (a `switch` over `self`
inside the property). Then write an enum `PurchaseResult` with associated
values for `.success(item: VendingItem, change: Double)` and
`.failure(reason: String)`, and a function `purchase(item:, inserted:) ->
PurchaseResult` that returns `.failure` if `inserted` is less than the
item's price, otherwise `.success` with the correct change.
