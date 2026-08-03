# 02 · Optionals Advanced

Level 1 introduced `Optional` as Swift's answer to "this value might be
missing," along with `if let` and basic unwrapping. In real code, optionals
show up constantly and chain into each other — parsing user input, walking
nested JSON, looking things up in dictionaries. This module covers the tools
that make working with them concise and, more importantly, safe.

## Optional chaining

Optional chaining lets you call methods, access properties, and subscript
through a chain of optionals — if any link in the chain is `nil`, the whole
expression short-circuits to `nil` instead of crashing:

```swift
struct Address {
    var street: String
    var apartmentNumber: String?
}

struct Person {
    var name: String
    var address: Address?
}

let alice = Person(name: "Alice", address: Address(street: "5th Ave", apartmentNumber: "12B"))
let bob = Person(name: "Bob", address: nil)

// "?." at each step: if address is nil, the whole expression is nil
print(alice.address?.apartmentNumber)   // Optional("12B")
print(bob.address?.apartmentNumber)     // nil

// Chaining through a method call
let uppercasedApt = alice.address?.apartmentNumber?.uppercased()
print(uppercasedApt)   // Optional("12B")
```

Notice the result is still wrapped in `Optional` — chaining doesn't unwrap
anything, it just protects you from crashing on a `nil` partway through.

## Nil-coalescing (`??`)

`??` supplies a default value when the left side is `nil`, unwrapping in the
same expression:

```swift
let aptOrDefault = alice.address?.apartmentNumber ?? "No apartment"
let bobAptOrDefault = bob.address?.apartmentNumber ?? "No apartment"
print(aptOrDefault)       // 12B
print(bobAptOrDefault)    // No apartment
```

`??` chains too, trying each fallback in order until one is non-nil:

```swift
let preferredName: String? = nil
let nickname: String? = nil
let fallbackName = "Guest"

let displayName = preferredName ?? nickname ?? fallbackName
print(displayName)   // Guest
```

## `guard let` for early exits

`if let` is fine for a quick conditional branch, but when a function needs
several values to proceed and should bail out immediately if any are
missing, `guard let` reads much better — it keeps the "happy path"
unindented and puts the failure case right where the check happens:

```swift
import Foundation

func formatBalance(from input: [String: String]) -> String {
    guard let rawAmount = input["amount"], let amount = Double(rawAmount) else {
        return "Invalid amount"
    }
    guard amount >= 0 else {
        return "Amount cannot be negative"
    }
    return String(format: "$%.2f", amount)
}

print(formatBalance(from: ["amount": "42.5"]))    // $42.50
print(formatBalance(from: ["amount": "oops"]))     // Invalid amount
print(formatBalance(from: ["amount": "-5"]))       // Amount cannot be negative
```

Every `guard` statement's `else` branch **must exit the current scope** —
`return`, `break`, `continue`, or `throw`. The compiler enforces this, which
is what guarantees `rawAmount` and `amount` are safely unwrapped for the
rest of the function.

## `map` and `flatMap` on `Optional`

`Optional` itself has `map` and `flatMap`, mirroring the array versions —
they let you transform the wrapped value without manually unwrapping first:

```swift
let input: String? = "42"

// map: transform the value if present, otherwise stay nil
let doubled = input.map { Int($0)! * 2 }   // careful: force-unwrap inside map is still risky
print(doubled)   // Optional(84)

// A safer version: the transform itself returns an optional
let asInt = input.flatMap { Int($0) }
print(asInt)   // Optional(42)

let badInput: String? = "not a number"
let badResult = badInput.flatMap { Int($0) }
print(badResult)   // nil
```

The difference: `map`'s transform returns a plain value, so `Optional.map`
wraps the result in `Optional` for you. If the transform *itself* returns an
`Optional` (like `Int.init?(String)`), plain `map` would give you a
double-wrapped `Int??` — `flatMap` flattens that back down to a single
`Int?`. This is the same reasoning as `Array.flatMap` from Level 1, applied
to a container that holds at most one element.

## Force-unwrapping: when it's fine, and when it isn't

`!` unwraps an optional immediately and crashes if it's `nil`. It has a bad
reputation, but it's not always wrong — it's wrong when the compiler can't
actually prove the value is safe:

```swift
// Reasonable: you just checked, or the shape of the data guarantees it
let numbers = [1, 2, 3]
let first = numbers.first!   // fine -- you can see the array is non-empty right above

// Dangerous: input you don't control
func parseAge(_ text: String) -> Int {
    return Int(text)!   // CRASHES if text isn't a valid integer, e.g. "abc" or ""
}
// parseAge("thirty")   // fatal error: unexpectedly found nil while unwrapping an Optional value
```

The rule of thumb: force-unwrap only when *you* can see, right there in the
code, why the value can never be `nil` at that point — never on values that
came from user input, network responses, or file contents. For everything
else, use `guard let`, `if let`, or `??`.

## `as?` and optional casting

Downcasting also produces an optional — `as?` returns `nil` instead of
crashing when the cast fails, and `as!` force-casts (with the same crash
risk as `!`):

```swift
let values: [Any] = [1, "two", 3.0, "four"]

for value in values {
    if let text = value as? String {
        print("String: \(text)")
    } else if let number = value as? Int {
        print("Int: \(number)")
    }
}
// Int: 1
// String: two
// String: four
```

`3.0` (a `Double`) matches neither `as? String` nor `as? Int`, so it's
silently skipped — exactly the safe-by-default behavior `as?` is for.

## Cheat sheet

| Tool | What it does | Crashes on `nil`? |
|------|---------------|--------------------|
| `?.` (optional chaining) | Access through an optional, short-circuits to `nil` | No |
| `??` | Supply a default, unwrapping in one step | No |
| `guard let ... else { return }` | Unwrap or exit the current scope early | No |
| `if let` | Unwrap into a conditional branch | No |
| `.map { }` | Transform the wrapped value if present | No |
| `.flatMap { }` | Transform with a closure that itself returns an optional, avoiding double-wrapping | No |
| `as?` | Attempt a downcast, `nil` on failure | No |
| `!` | Force-unwrap | **Yes**, if `nil` |
| `as!` | Force-downcast | **Yes**, if the cast fails |

## Exercise

Write a function `firstValidEmail(_ candidates: [String?]) -> String?` that
takes an array of optional strings and returns the first one that both
exists and contains `"@"` — do it using `compactMap` and `first(where:)`
rather than a manual loop. Then write `extractDomain(from email: String) ->
String?` using `flatMap`-style chaining on the result of `email.split(separator:
"@").last` to pull out the part after the `@`, returning `nil` for malformed
input. Test both with a mix of `nil`, valid, and invalid entries, printing
results with `??` fallbacks instead of force-unwrapping.
