# 09 · Closures Intro

A closure is a self-contained block of functionality that can be passed
around and called later — in fact, every function you've written so far is
a closure with a name. Swift closures can also capture variables from the
surrounding scope where they're defined.

## Closure syntax, step by step

```swift
// Full syntax
let fullAdd: (Int, Int) -> Int = { (a: Int, b: Int) -> Int in
    return a + b
}
print(fullAdd(2, 3))   // 5

// Types can be inferred from context (the variable's declared type)
let inferredAdd: (Int, Int) -> Int = { a, b in
    return a + b
}
print(inferredAdd(2, 3))   // 5

// Single-expression closures can omit "return"
let implicitReturn: (Int, Int) -> Int = { a, b in a + b }
print(implicitReturn(2, 3))   // 5

// Shorthand argument names $0, $1, ... for very short closures
let shorthand: (Int, Int) -> Int = { $0 + $1 }
print(shorthand(2, 3))   // 5
```

All four of the above do exactly the same thing — real Swift code typically
lands somewhere between the "types inferred" and "shorthand" styles,
depending on how self-explanatory the closure is.

## Closures as function arguments

```swift
func applyOperation(_ a: Int, _ b: Int, using operation: (Int, Int) -> Int) -> Int {
    return operation(a, b)
}

let sum = applyOperation(4, 5, using: { a, b in a + b })
print(sum)   // 9

let product = applyOperation(4, 5) { a, b in a * b }   // trailing closure syntax
print(product)   // 20
```

**Trailing closure syntax**: when a closure is the last argument, it can be
written outside the parentheses — this is idiomatic Swift and shows up
constantly with the standard library's `map`/`filter`/`sorted`.

## Sorting with closures

```swift
let names = ["Charlie", "Alice", "bob"]

let sortedNames = names.sorted { first, second in
    first.lowercased() < second.lowercased()
}
print(sortedNames)   // ["Alice", "bob", "Charlie"]

let sortedByLength = names.sorted { $0.count < $1.count }
print(sortedByLength)   // ["bob", "Alice", "Charlie"]
```

## Capturing values from the surrounding scope

A closure "closes over" (captures) constants and variables from the context
where it's created, and can keep using them even after that context is gone.

```swift
func makeIncrementer(incrementAmount: Int) -> () -> Int {
    var total = 0
    let incrementer: () -> Int = {
        total += incrementAmount   // captures "total" and "incrementAmount"
        return total
    }
    return incrementer
}

let incrementByTwo = makeIncrementer(incrementAmount: 2)
print(incrementByTwo())   // 2
print(incrementByTwo())   // 4
print(incrementByTwo())   // 6

let incrementByTen = makeIncrementer(incrementAmount: 10)
print(incrementByTen())   // 10 -- an entirely separate "total"
print(incrementByTwo())   // 8  -- incrementByTwo's "total" is unaffected
```

Each call to `makeIncrementer` creates a fresh `total` variable, and the
returned closure keeps its own private reference to it — closures capture by
reference, so `total` stays alive exactly as long as something still refers
to it.

## Closures with `map`, `filter`, `reduce`, revisited

```swift
let temperaturesCelsius = [0, 20, 30, -10, 100]

let fahrenheit = temperaturesCelsius.map { celsius in
    Double(celsius) * 9 / 5 + 32
}
print(fahrenheit)   // [32.0, 68.0, 86.0, 14.0, 212.0]

let freezing = temperaturesCelsius.filter { $0 <= 0 }
print(freezing)   // [0, -10]

let hottest = temperaturesCelsius.reduce(Int.min) { currentMax, next in
    max(currentMax, next)
}
print(hottest)   // 100
```

## Cheat sheet

| Style | Example |
|-------|---------|
| Full syntax | `{ (a: Int, b: Int) -> Int in return a + b }` |
| Inferred types | `{ a, b in return a + b }` |
| Implicit return | `{ a, b in a + b }` |
| Shorthand args | `{ $0 + $1 }` |
| Trailing closure | `numbers.sorted { $0 < $1 }` |

## Exercise

Write a function `makeMultiplier(factor: Int) -> (Int) -> Int` that returns a
closure multiplying its input by `factor` (using the closure-capture pattern
from `makeIncrementer` above). Then, given
`let words = ["banana", "kiwi", "apple", "fig"]`, use trailing closure syntax
with `sorted` to sort them by length, and with `filter` + `map` to produce an
uppercased array of only the words longer than 3 letters.
