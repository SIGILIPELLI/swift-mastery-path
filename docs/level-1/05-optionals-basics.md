# 05 · Optionals Basics

## The problem optionals solve

In many languages, any reference can silently be `null`, and the compiler
doesn't remind you to check. Swift makes the *possible absence* of a value
part of the type system: a plain `String` is guaranteed to hold a string; an
`String?` ("optional String") might hold a string, or might hold nothing.

```swift
var name: String? = "Ada"
name = nil   // legal -- name is optional, so it can hold "no value"

// var required: String = "Ada"
// required = nil   // compile error -- non-optional types can never be nil
```

## Declaring and unwrapping with `!` (force unwrap)

```swift
let possibleNumber: String = "42"
let convertedNumber: Int? = Int(possibleNumber)   // Int("42") returns Int?

print(convertedNumber!)   // 42 -- force-unwrap: "I'm certain this isn't nil"

let badNumber: Int? = Int("not a number")
// print(badNumber!)   // would crash at runtime: "Fatal error: Unexpectedly found nil"
```

Force-unwrapping with `!` is a promise to the compiler that you've already
ruled out `nil` — if you're wrong, the program crashes immediately. Treat `!`
as a last resort, not a habit.

## Safe unwrapping with `if let`

```swift
let possibleNumber: String = "abc"
let convertedNumber = Int(possibleNumber)

if let actualNumber = convertedNumber {
    print("It's a number: \(actualNumber)")
} else {
    print("Not a valid number")
}
// Not a valid number
```

Since Swift 5.7, when the unwrapped constant has the same name as the
optional, you can shorten this:

```swift
let convertedNumber: Int? = Int("42")

if let convertedNumber {
    print("Got \(convertedNumber)")   // Got 42
}
```

## `guard let` — early exit

`guard` reads like an "or else, bail out" statement, and is the idiomatic
choice inside functions: it keeps the happy path unindented and unwraps the
value for use in the *rest* of the function, not just an inner block.

```swift
func greet(_ name: String?) -> String {
    guard let name else {
        return "Hello, stranger!"
    }
    // "name" is a non-optional String from here to the end of the function
    return "Hello, \(name)!"
}

print(greet("Grace"))   // Hello, Grace!
print(greet(nil))        // Hello, stranger!
```

## Nil-coalescing operator `??`

Provide a default value to fall back to when an optional is `nil`, in a
single expression:

```swift
let storedName: String? = nil
let displayName = storedName ?? "Guest"
print(displayName)   // Guest

let savedScore: Int? = 95
let effectiveScore = savedScore ?? 0
print(effectiveScore)   // 95
```

## Optional chaining

Access properties/methods through an optional with `?.` — if any link in the
chain is `nil`, the whole expression short-circuits to `nil` instead of
crashing.

```swift
struct Address {
    let city: String
}
struct Person {
    let name: String
    let address: Address?
}

let personWithAddress = Person(name: "Ada", address: Address(city: "London"))
let personWithoutAddress = Person(name: "Alan", address: nil)

print(personWithAddress.address?.city)      // Optional("London")
print(personWithoutAddress.address?.city)   // nil

// Combine with ?? for a plain, non-optional result:
print(personWithoutAddress.address?.city ?? "Unknown")   // Unknown
```

## Implicitly unwrapped optionals

`String!` behaves like `String?` but is auto-unwrapped on access, without
`!` at each use site. Reserved for rare cases (like properties set right
after initialization but before first use) — regular optionals are almost
always the better choice.

```swift
var window: String! = "Main Window"
print(window.count)   // used directly, no "!" or "?" needed -- 11
```

## Cheat sheet

| Technique | Syntax | When to use |
|-----------|--------|--------------|
| Force unwrap | `value!` | Only when `nil` is truly impossible; crashes otherwise |
| Optional binding | `if let x = value { ... }` | Use the value only inside this block |
| Early exit | `guard let x = value else { return }` | Use the value for the rest of the function |
| Default value | `value ?? defaultValue` | One-line fallback, no branching needed |
| Optional chaining | `value?.property` | Safely reach through a chain of optionals |
| Implicitly unwrapped | `String!` | Rare; auto-unwraps, still crashes if `nil` |

## Exercise

Write a function `parseAge(_ input: String) -> Int?` that uses `Int(input)`
to attempt a conversion. Then write `describeAge(_ input: String) -> String`
that calls `parseAge`, uses `guard let` to bail out with `"Invalid age"` if
parsing fails, and otherwise returns `"You are N years old"` — where ages
under 0 or over 150 should also count as invalid.
