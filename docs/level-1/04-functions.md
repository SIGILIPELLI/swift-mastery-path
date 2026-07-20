# 04 · Functions

## Basic declaration

```swift
func greet(name: String) -> String {
    return "Hello, \(name)!"
}

print(greet(name: "Ada"))   // Hello, Ada!
```

A function with no return value can omit `-> Type` entirely (it implicitly
returns `Void`, i.e. `()`):

```swift
func logMessage(_ message: String) {
    print("[LOG] \(message)")
}

logMessage("Server started")   // [LOG] Server started
```

## Argument labels vs. parameter names

Swift functions have two names per parameter: an **argument label** used at
the call site, and a **parameter name** used inside the function body. By
default they're the same, but you can customize either.

```swift
// external label "to", internal name "recipient"
func send(to recipient: String, message: String) {
    print("Sending '\(message)' to \(recipient)")
}
send(to: "Grace", message: "Hi!")
// Sending 'Hi!' to Grace

// underscore "_" drops the label entirely at the call site
func multiply(_ a: Int, _ b: Int) -> Int {
    return a * b
}
print(multiply(3, 4))   // 12
```

This is why `greet(name:)` reads like a sentence at the call site
(`greet(name: "Ada")`) while `multiply(_:_:)` reads like ordinary math
(`multiply(3, 4)`) — the label design is a deliberate part of the API.

## Default parameter values

```swift
func makeGreeting(name: String, greeting: String = "Hello") -> String {
    return "\(greeting), \(name)!"
}

print(makeGreeting(name: "Sam"))                    // Hello, Sam!
print(makeGreeting(name: "Sam", greeting: "Hey"))    // Hey, Sam!
```

## Variadic parameters

```swift
func sum(_ numbers: Int...) -> Int {
    var total = 0
    for n in numbers {
        total += n
    }
    return total
}

print(sum(1, 2, 3))       // 6
print(sum(10, 20, 30, 40)) // 100
print(sum())               // 0
```

Inside the function, `numbers` is just an `[Int]` array.

## Multiple return values with tuples

```swift
func minMax(_ values: [Int]) -> (min: Int, max: Int)? {
    guard let first = values.first else { return nil }
    var currentMin = first
    var currentMax = first
    for value in values[1...] where !values.isEmpty {
        if value < currentMin { currentMin = value }
        if value > currentMax { currentMax = value }
    }
    return (currentMin, currentMax)
}

if let result = minMax([8, 3, 15, 1, 9]) {
    print("min: \(result.min), max: \(result.max)")
}
// min: 1, max: 15
```

## `inout` parameters

By default, arguments are passed by value — modifying a parameter inside a
function doesn't affect the caller's variable. `inout` opts a parameter into
being mutated in place.

```swift
func doubleInPlace(_ value: inout Int) {
    value *= 2
}

var number = 21
doubleInPlace(&number)   // "&" is required at the call site
print(number)             // 42
```

## Functions as types, and nested functions

A function's type is its parameter types plus its return type, e.g.
`(Int, Int) -> Int`. Functions can be assigned to variables, passed as
arguments, and nested inside other functions.

```swift
func add(_ a: Int, _ b: Int) -> Int { a + b }
func subtract(_ a: Int, _ b: Int) -> Int { a - b }

var operation: (Int, Int) -> Int = add
print(operation(5, 3))   // 8
operation = subtract
print(operation(5, 3))   // 2

func chooseOperation(addMode: Bool) -> (Int, Int) -> Int {
    func adder(_ a: Int, _ b: Int) -> Int { a + b }
    func subtracter(_ a: Int, _ b: Int) -> Int { a - b }
    return addMode ? adder : subtracter
}

let op = chooseOperation(addMode: true)
print(op(10, 4))   // 14
```

## Cheat sheet

| Feature | Syntax | Notes |
|---------|--------|-------|
| Argument label | `func f(label name: Type)` | Label used at call site, `name` used inside |
| Drop label | `func f(_ name: Type)` | No label required at call site |
| Default value | `func f(x: Int = 0)` | Caller may omit the argument |
| Variadic | `func f(_ xs: Int...)` | Becomes `[Int]` inside the function |
| Multiple returns | `-> (a: Int, b: String)` | Tuple return type, optionally labeled |
| Mutate caller's variable | `func f(_ x: inout Int)` | Call with `f(&value)` |
| Function type | `(Int, Int) -> Int` | Functions can be stored, passed, returned |

## Exercise

Write a function `describe(number:)` that takes an `Int` and returns a
`String` describing whether it's negative, zero, or positive, and whether
it's even or odd (e.g. `"positive and even"`). Then write a function
`applyTwice(_:to:)` that takes a function `(Int) -> Int` and an `Int`, and
applies the function to the value twice (e.g. `applyTwice({ $0 * 2 }, to: 3)`
returns `12`).
