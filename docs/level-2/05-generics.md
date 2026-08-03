# 05 · Generics

You've already been using generics without necessarily calling them that —
`Array<Element>`, `Optional<Wrapped>`, and `Dictionary<Key, Value>` are all
generic types from the standard library. Generics let you write one function
or type that works with *any* type, while still keeping full type safety —
no casting, no `Any`, no runtime surprises.

## The problem generics solve

Without generics, supporting multiple types means duplicating code:

```swift
func swapInts(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

func swapStrings(_ a: inout String, _ b: inout String) {
    let temp = a
    a = b
    b = temp
}
```

The logic is identical — only the type differs. A generic function
expresses that once:

```swift
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var x = 1, y = 2
swapValues(&x, &y)
print(x, y)   // 2 1

var name1 = "Alice", name2 = "Bob"
swapValues(&name1, &name2)
print(name1, name2)   // Bob Alice
```

`T` is a **type parameter** — a placeholder the compiler fills in with a
concrete type at each call site. `swapValues(&x, &y)` compiles as if you'd
written `swapValues<Int>`, and `swapValues(&name1, &name2)` as if you'd
written `swapValues<String>` — both fully type-checked, with zero runtime
overhead for the abstraction.

## Generic functions with return values

```swift
func firstAndLast<T>(_ items: [T]) -> (first: T, last: T)? {
    guard let first = items.first, let last = items.last else {
        return nil
    }
    return (first, last)
}

if let result = firstAndLast([10, 20, 30]) {
    print(result.first, result.last)   // 10 30
}

if let result = firstAndLast(["x", "y", "z"]) {
    print(result.first, result.last)   // x z
}

print(firstAndLast([Int]()) == nil)   // true -- empty array
```

## Type constraints

Sometimes `T` can be *anything*, but sometimes you need `T` to support a
specific capability — like being comparable with `<`. A **constraint**
requires `T` to conform to a protocol:

```swift
func largest<T: Comparable>(_ items: [T]) -> T? {
    guard var current = items.first else { return nil }
    for item in items.dropFirst() where item > current {
        current = item
    }
    return current
}

print(largest([3, 7, 2, 9, 4]) ?? "empty")     // 9
print(largest(["pear", "apple", "kiwi"]) ?? "empty")   // pear
```

Without `T: Comparable`, the line `item > current` wouldn't compile — the
compiler has no way to know an arbitrary `T` supports `>`. The constraint is
what makes `>` legal, by guaranteeing every `T` passed in has it.

## Generic types

Types, not just functions, can be generic — this is how you'd build your
own `Stack`, `Queue`, or `Box`:

```swift
struct Stack<Element> {
    private var items: [Element] = []

    mutating func push(_ item: Element) {
        items.append(item)
    }

    mutating func pop() -> Element? {
        items.popLast()
    }

    var isEmpty: Bool { items.isEmpty }
    var count: Int { items.count }
}

var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
intStack.push(3)
print(intStack.pop() ?? -1)   // 3
print(intStack.count)          // 2

var stringStack = Stack<String>()
stringStack.push("first")
stringStack.push("second")
print(stringStack.pop() ?? "empty")   // second
```

`Stack<Int>` and `Stack<String>` are different **specializations** of the
same generic type — the compiler generates type-safe code for each, so
`intStack.push("oops")` is a compile error, not a runtime one.

## Multiple constraints and `where` clauses

You can require more than one capability, and use `where` for more complex
constraints, including constraints on associated types:

```swift
func printSortedUnique<T: Comparable & Hashable>(_ items: [T]) {
    let unique = Set(items)          // needs Hashable
    let sorted = unique.sorted()      // needs Comparable
    print(sorted)
}

printSortedUnique([3, 1, 2, 3, 1])   // [1, 2, 3]

func allElementsMatch<C1: Collection, C2: Collection>(_ first: C1, _ second: C2) -> Bool
    where C1.Element == C2.Element, C1.Element: Equatable {
    guard first.count == second.count else { return false }
    return zip(first, second).allSatisfy { $0 == $1 }
}

print(allElementsMatch([1, 2, 3], [1, 2, 3]))        // true
print(allElementsMatch([1, 2, 3], Set([3, 2, 1])))   // true -- Array and Set, same elements
```

`allElementsMatch` accepts two *different* collection types (`Array` and
`Set` in the second call) as long as they hold the same, `Equatable`
element type — generics compose with protocols to express exactly the
requirement you need, no more and no less.

## Associated types: generics in protocols

A protocol can declare a placeholder type using `associatedtype` — the
conforming type fills it in, similar to how a generic function's caller
fills in `T`:

```swift
protocol Container {
    associatedtype Item
    mutating func add(_ item: Item)
    var count: Int { get }
}

struct IntBag: Container {
    private var items: [Int] = []
    mutating func add(_ item: Int) { items.append(item) }   // Item is inferred as Int
    var count: Int { items.count }
}

var bag = IntBag()
bag.add(5)
bag.add(10)
print(bag.count)   // 2
```

This is exactly how the standard library's own `Sequence` and `Collection`
protocols work under the hood — `associatedtype Element` is why `for x in
myArray` and `for x in mySet` both work with fully type-safe `x`.

## Cheat sheet

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Generic function | `func f<T>(_ x: T)` | One implementation, many concrete types |
| Type constraint | `<T: Comparable>` | Require a capability so the body can use it |
| Multiple constraints | `<T: Comparable & Hashable>` | Require more than one capability |
| `where` clause | `where C1.Element == C2.Element` | Constrain relationships between type parameters |
| Generic type | `struct Stack<Element> { ... }` | A reusable container/type, specialized per use |
| `associatedtype` | `protocol Container { associatedtype Item }` | A placeholder type inside a protocol |

## Exercise

Write a generic function `func average<T: BinaryInteger>(_ values: [T]) ->
Double` that returns the arithmetic mean as a `Double` (hint: convert each
element with `Double(value)`), returning `0` for an empty array. Then define
a generic `struct Pair<A, B>` holding a `first: A` and `second: B`, with a
method `swapped() -> Pair<B, A>` that returns a new pair with the elements
reversed. Create a `Pair<String, Int>`, print it, call `swapped()`, and print
the result to confirm the types flipped correctly.
