# 06 · Collections

Swift has three primary collection types, all **value types**: `Array`
(ordered), `Set` (unordered, unique elements), and `Dictionary` (key-value
pairs). Assigning one to a new variable or passing it to a function copies
it (conceptually — Swift optimizes this with copy-on-write under the hood).

## Arrays

```swift
var fruits = ["apple", "banana", "cherry"]
print(fruits.count)        // 3
print(fruits[0])            // apple

fruits.append("date")
fruits += ["elderberry"]
print(fruits)                // ["apple", "banana", "cherry", "date", "elderberry"]

fruits.remove(at: 1)         // removes "banana"
print(fruits)                // ["apple", "cherry", "date", "elderberry"]

let empty: [Int] = []
var explicit: Array<Int> = [1, 2, 3]   // "[Int]" is shorthand for "Array<Int>"

for fruit in fruits {
    print(fruit, terminator: " ")
}
// apple cherry date elderberry
print()
```

Common transformations use `map`, `filter`, and `reduce` rather than manual
loops:

```swift
let numbers = [1, 2, 3, 4, 5, 6]

let doubled = numbers.map { $0 * 2 }
print(doubled)   // [2, 4, 6, 8, 10, 12]

let evens = numbers.filter { $0 % 2 == 0 }
print(evens)   // [2, 4, 6]

let total = numbers.reduce(0) { $0 + $1 }
print(total)   // 21

let sorted = numbers.sorted(by: >)
print(sorted)   // [6, 5, 4, 3, 2, 1]
```

## Dictionaries

```swift
var ages: [String: Int] = ["Ada": 36, "Alan": 41]
ages["Grace"] = 85          // insert
ages["Ada"] = 37             // update

print(ages["Ada"]!)          // 37 -- subscripting a Dictionary returns an Optional
print(ages["Unknown"])       // nil

if let age = ages["Alan"] {
    print("Alan is \(age)")   // Alan is 41
}

ages.removeValue(forKey: "Grace")

for (name, age) in ages {
    print("\(name) is \(age)")   // order is not guaranteed
}

let defaultedAge = ages["Nobody", default: 0]
print(defaultedAge)   // 0
```

Dictionary subscripting always returns an `Optional` (`Value?`), because the
key might not exist — this is the same optional mechanism from
[Module 5](05-optionals-basics.md), applied consistently across the language.

## Sets

```swift
var primes: Set<Int> = [2, 3, 5, 7, 11]
primes.insert(13)
primes.insert(2)              // no-op -- 2 is already present
print(primes.contains(7))     // true
print(primes.count)            // 6

let a: Set = [1, 2, 3, 4]
let b: Set = [3, 4, 5, 6]

print(a.union(b).sorted())          // [1, 2, 3, 4, 5, 6]
print(a.intersection(b).sorted())   // [3, 4]
print(a.subtracting(b).sorted())    // [1, 2]
```

Sets guarantee uniqueness and offer fast `contains` checks (O(1) on average),
at the cost of not preserving insertion order.

## Choosing between them

| Type | Ordered? | Duplicates? | Lookup by | Typical use |
|------|----------|-------------|-----------|-------------|
| `Array` | Yes | Yes | Index | Sequential data, order matters |
| `Set` | No | No | Value (hash) | Uniqueness, fast membership checks |
| `Dictionary` | No | Keys unique | Key | Fast lookup by identifier |

## Nested collections

```swift
let matrix: [[Int]] = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
]

for row in matrix {
    print(row.map(String.init).joined(separator: " "))
}
// 1 2 3
// 4 5 6
// 7 8 9

let studentsBySubject: [String: [String]] = [
    "Math": ["Ada", "Alan"],
    "Physics": ["Grace"],
]
print(studentsBySubject["Math"] ?? [])   // ["Ada", "Alan"]
```

## Exercise

Given `let words = ["swift", "is", "expressive", "and", "safe"]`, use `map`
to produce an array of their lengths, `filter` to keep only words with more
than 3 characters, and `reduce` to compute the total character count across
all words. Then build a `[String: Int]` dictionary mapping each word to its
length, and a `Set<Int>` of the distinct lengths.
