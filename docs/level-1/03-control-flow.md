# 03 · Control Flow

## `if` / `else`

```swift
let temperature = 72

if temperature > 80 {
    print("It's hot")
} else if temperature > 60 {
    print("It's nice out")
} else {
    print("Bring a jacket")
}
// It's nice out
```

Unlike C-family languages, Swift requires the condition to be a `Bool` —
there's no implicit conversion from `Int` to `Bool`, so `if temperature`
(without a comparison) simply won't compile.

## `if` as an expression

Since Swift 5.9, `if`/`switch` can produce a value directly when every branch
yields the same type:

```swift
let temperature = 55
let advice = if temperature > 80 {
    "wear shorts"
} else if temperature > 60 {
    "wear a t-shirt"
} else {
    "wear a coat"
}
print(advice)   // wear a coat
```

## `switch`

Swift's `switch` is far more powerful than C's — it requires exhaustiveness,
never falls through by default, and can match ranges, tuples, and patterns.

```swift
let httpStatus = 404

switch httpStatus {
case 200:
    print("OK")
case 301, 302:
    print("Redirect")
case 400...499:
    print("Client error")     // matches the range 400-499
case 500...599:
    print("Server error")
default:
    print("Unknown status")
}
// Client error
```

```swift
let point = (x: 0, y: 5)

switch point {
case (0, 0):
    print("Origin")
case (0, _):
    print("On the y-axis")     // "_" ignores that part of the tuple
case (_, 0):
    print("On the x-axis")
default:
    print("Somewhere else")
}
// On the y-axis
```

Adding `where` filters a case further:

```swift
let score = 85

switch score {
case let s where s >= 90:
    print("A")
case let s where s >= 80:
    print("B")
default:
    print("C or below")
}
// B
```

## `for-in` loops

```swift
for i in 1...5 {
    print(i, terminator: " ")
}
// 1 2 3 4 5
print()

for i in stride(from: 0, to: 10, by: 2) {
    print(i, terminator: " ")
}
// 0 2 4 6 8
print()

let names = ["Ada", "Grace", "Alan"]
for name in names {
    print("Hello, \(name)")
}
// Hello, Ada
// Hello, Grace
// Hello, Alan
```

`1...5` is a *closed range* (includes 5); `0..<10` is a *half-open range*
(excludes 10) — the more common choice when iterating array indices.

## `while` and `repeat-while`

```swift
var countdown = 3
while countdown > 0 {
    print(countdown)
    countdown -= 1
}
print("Liftoff!")
// 3
// 2
// 1
// Liftoff!

var attempts = 0
repeat {
    attempts += 1
    print("Attempt \(attempts)")
} while attempts < 3
// Attempt 1
// Attempt 2
// Attempt 3
```

`repeat-while` is Swift's version of `do-while` — the body always runs at
least once, since the condition is checked after.

## `break`, `continue`, and labeled loops

```swift
outer: for i in 1...3 {
    for j in 1...3 {
        if j == 2 {
            continue outer   // skip to the next value of i
        }
        if i == 3 {
            break outer       // exit both loops entirely
        }
        print("i=\(i) j=\(j)")
    }
}
// i=1 j=1
// i=2 j=1
```

Labels (`outer:`) let `break`/`continue` target an outer loop directly,
instead of only the innermost one.

## Cheat sheet

| Construct | Use when |
|-----------|----------|
| `if`/`else if`/`else` | Branching on boolean conditions |
| `switch` | Matching many discrete values, ranges, or patterns exhaustively |
| `for-in` | Iterating a known sequence, range, or collection |
| `while` | Looping while a condition holds, unknown iteration count |
| `repeat-while` | Same as `while`, but body always runs once first |
| `break` / `continue` | Exiting or skipping an iteration; add a label to target an outer loop |

## Exercise

Write a program that loops from 1 to 30. For each number, print `"Fizz"` if
divisible by 3, `"Buzz"` if divisible by 5, `"FizzBuzz"` if divisible by both,
and the number itself otherwise — implement it once with `if/else` and once
with `switch` using `where` clauses.
