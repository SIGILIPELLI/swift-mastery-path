# 02 · Variables & Types

## `let` vs `var`

Swift distinguishes constants from variables at the language level, and
defaults to immutability — you opt into mutability, not the other way around.

```swift
let name = "Ada"       // constant -- cannot be reassigned
var score = 0          // variable -- can be reassigned

score = 10
score += 5
print(score)            // 15

// name = "Grace"       // compile error: cannot assign to a 'let' constant
```

Prefer `let` by default. Only reach for `var` when a value genuinely needs to
change after creation — the compiler will even suggest changing a `var` to
`let` if you never mutate it.

## Type inference and explicit annotations

Swift infers types from the initial value, but you can annotate explicitly
when there's no initializer yet or you want a wider/narrower type than the
default.

```swift
let inferredInt = 42            // Int
let inferredDouble = 3.14       // Double
let inferredString = "hello"    // String
let inferredBool = true         // Bool

var declaredFirst: Int          // no value yet -- must annotate the type
declaredFirst = 100

let smallNumber: Int8 = 127     // explicit narrower integer type
let pi: Float = 3.14159         // explicit narrower floating-point type
```

## Basic types

```swift
let age: Int = 30                 // whole numbers (word-sized: Int64 on 64-bit platforms)
let temperature: Double = 98.6    // 64-bit floating point (the default for decimals)
let ratio: Float = 0.5            // 32-bit floating point
let initial: Character = "A"      // a single grapheme cluster
let greeting: String = "Hi there" // a sequence of characters
let isActive: Bool = true         // true or false
```

`Int` and `Double` are the defaults you should reach for unless you have a
specific reason (interop with a C API, memory-constrained storage) to pick a
sized variant like `Int8`, `UInt32`, or `Float`.

## Type safety and conversions

Swift never implicitly converts between types — even `Int` and `Double`
require an explicit conversion.

```swift
let count = 5          // Int
let price = 2.99        // Double

// let total = count * price   // compile error: Int and Double don't mix

let total = Double(count) * price   // explicit conversion
print(total)                         // 14.950000000000001

let rounded = Int(total)              // truncates toward zero
print(rounded)                        // 14
```

## String interpolation

Embed expressions directly inside string literals with `\(...)`:

```swift
let user = "Sam"
let visits = 3
print("\(user) has visited \(visits) times")
// Sam has visited 3 times

print("Next visit will be number \(visits + 1)")
// Next visit will be number 4
```

## Type aliases

`typealias` gives an existing type a second, more descriptive name — purely
for readability, no new type is created.

```swift
typealias Meters = Double

let trackLength: Meters = 400.0
print(trackLength)   // 400.0
```

## Tuples

A tuple groups multiple values into one compound value, without needing a
named type.

```swift
let httpResponse = (status: 200, message: "OK")
print(httpResponse.status)    // 200
print(httpResponse.message)   // OK

let (code, text) = httpResponse   // destructuring
print("\(code): \(text)")          // 200: OK
```

## Cheat sheet

| Type | Example | Notes |
|------|---------|-------|
| `Int` | `let x = 5` | Default integer type, platform word size |
| `Double` | `let x = 5.0` | Default floating-point type |
| `Float` | `let x: Float = 5.0` | 32-bit, smaller/less precise than `Double` |
| `String` | `let x = "hi"` | Unicode-correct, value type |
| `Character` | `let x: Character = "A"` | A single grapheme cluster |
| `Bool` | `let x = true` | `true`/`false` only, no truthy/falsy values |
| `(T, U, ...)` | `let x = (1, "a")` | Tuple — compound, unnamed type |

## Exercise

Declare constants for a person's `name` (String), `age` (Int), and `height`
(Double, in meters). Use string interpolation to print a single sentence
combining all three. Then create a tuple `(min: Int, max: Int)` representing
a temperature range, destructure it into two named variables, and print both.
