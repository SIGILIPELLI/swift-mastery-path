---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Swift Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```swift\nstruct Box { var items: [Int] }\nvar a = Box(items: [1])\nvar b = a\nb.items.append(2)\nfinal class Ref { var items: [Int] = [1] }\nlet r1 = Ref(); let r2 = r1; r2.items.append(2)\nprint(a.items, b.items, r1.items, r1 === r2)\n```\n",
    "options": [
      "[1] [1, 2] [1, 2] true",
      "[1, 2] [1, 2] [1, 2] true",
      "[1] [1, 2] [1] false",
      "[1] [1] [1, 2] true"
    ],
    "correctIndex": 0,
    "explanation": "Structs copy on assignment; class instances are shared references. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```swift\nfunc parse(_ s: String) -> Int? { Int(s) }\nlet inputs = [\"4\", \"x\", \"7\"]\nlet nums = inputs.compactMap(parse)\nlet total = inputs.compactMap(parse).reduce(0, +)\nlet firstBad = inputs.first { parse($0) == nil } ?? \"none\"\nprint(nums, total, firstBad)\nlet d: [String: Int] = [\"a\": 1]\nprint(d[\"a\"] ?? -1, d[\"b\"] ?? -1)\n```\n",
    "options": [
      "[4, nil, 7] 11 x\n1 -1",
      "[4, 7] 11 x\n1 nil",
      "[4, 7] 11 none\n1 -1",
      "[4, 7] 11 x\n1 -1"
    ],
    "correctIndex": 3,
    "explanation": "compactMap drops nils and ?? gives defaults for missing keys. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```swift\nenum Cmd { case add(Int), mul(Int), reset }\nfunc apply(_ c: Cmd, to v: Int) -> Int {\n  switch c { case .add(let n): return v + n\n  case .mul(let n): return v * n\n  case .reset: return 0 }\n}\nlet result = [Cmd.add(3), .mul(4), .add(-2), .reset, .add(5)].reduce(0) { apply($1, to: $0) }\nprint(result)\n```\n",
    "options": [
      "15",
      "10",
      "5",
      "0"
    ],
    "correctIndex": 2,
    "explanation": "The reducer applies each command in order; reset discards everything before it. Verified by compiling and running it."
  },
  {
    "question": "Why is force-unwrapping (!) discouraged outside tests and truly impossible cases?",
    "options": [
      "It copies the value",
      "It is deprecated",
      "It crashes at runtime when the optional is nil",
      "It is slow"
    ],
    "correctIndex": 2,
    "explanation": "Use guard let, if let or ??."
  },
  {
    "question": "What does guard let do for readability?",
    "options": [
      "Catches",
      "Handles the failure case early and exits, so the main logic stays unindented",
      "Loops",
      "Casts"
    ],
    "correctIndex": 1,
    "explanation": "The else branch must leave the scope."
  },
  {
    "question": "What does a Swift switch require?",
    "options": [
      "Integer subjects",
      "To be exhaustive: every case handled or a default",
      "A default always",
      "Fallthrough"
    ],
    "correctIndex": 1,
    "explanation": "Adding an enum case then flags every switch."
  },
  {
    "question": "When do you reach for a class instead of a struct?",
    "options": [
      "Never",
      "For all data",
      "Always",
      "When you need shared identity, inheritance or reference semantics"
    ],
    "correctIndex": 3,
    "explanation": "Structs are the default."
  },
  {
    "question": "What creates a retain cycle with closures?",
    "options": [
      "Structs",
      "A closure stored by an object strongly capturing that object (self)",
      "Optionals",
      "Enums"
    ],
    "correctIndex": 1,
    "explanation": "Use [weak self] or [unowned self]."
  },
  {
    "question": "What does throws plus do/catch give you over returning nil?",
    "options": [
      "Nothing",
      "Speed",
      "Less code",
      "Specific error information the caller can handle differently"
    ],
    "correctIndex": 3,
    "explanation": "Optionals lose the reason."
  },
  {
    "question": "What does Codable synthesis give you?",
    "options": [
      "Persistence",
      "Automatic JSON (and other format) encoding and decoding for conforming types",
      "Validation",
      "Networking"
    ],
    "correctIndex": 1,
    "explanation": "Custom CodingKeys rename fields."
  },
  {
    "question": "What does await mark?",
    "options": [
      "A lock",
      "Blocking",
      "A possible suspension point where the task may yield its thread to other work",
      "A thread"
    ],
    "correctIndex": 2,
    "explanation": "Not a blocking call."
  },
  {
    "question": "What does an actor protect?",
    "options": [
      "Files",
      "Its mutable state, by serialising access to it",
      "Threads",
      "Network"
    ],
    "correctIndex": 1,
    "explanation": "Beware reentrancy at await points."
  },
  {
    "question": "What does structured concurrency guarantee about child tasks?",
    "options": [
      "They finish (or are cancelled) before their scope exits, and errors and cancellation propagate",
      "They are daemons",
      "They never fail",
      "They run forever"
    ],
    "correctIndex": 0,
    "explanation": "Task groups and async let."
  },
  {
    "question": "Where do you store tokens on iOS?",
    "options": [
      "A plist",
      "Files",
      "The Keychain, not UserDefaults",
      "UserDefaults"
    ],
    "correctIndex": 2,
    "explanation": "Choose an appropriate accessibility level."
  },
  {
    "question": "Why prefer protocols and dependency injection for network code?",
    "options": [
      "Speed",
      "Colour",
      "Law",
      "Tests can inject fakes and avoid real network calls"
    ],
    "correctIndex": 3,
    "explanation": "Deterministic tests."
  },
  {
    "question": "What does ARC do?",
    "options": [
      "Pools",
      "Garbage collects later",
      "Frees objects deterministically when their reference count reaches zero",
      "Manual free"
    ],
    "correctIndex": 2,
    "explanation": "Cycles need weak references."
  },
  {
    "question": "Why should heavy work stay off the main thread in an app?",
    "options": [
      "Law",
      "The main thread draws the UI, so blocking it freezes the interface",
      "Style",
      "Memory"
    ],
    "correctIndex": 1,
    "explanation": "Use Task or background actors."
  },
  {
    "question": "What does a multi-stage Dockerfile do for a server-side Swift service?",
    "options": [
      "Law",
      "Style",
      "Builds with the full toolchain and ships only the binary in a slim runtime image",
      "Speed only"
    ],
    "correctIndex": 2,
    "explanation": "Smaller attack surface."
  },
  {
    "question": "What proves a Swift project is production-ready?",
    "options": [
      "Looks good",
      "It runs",
      "CI with tests, linting, performance checks and operational readiness",
      "Few lines"
    ],
    "correctIndex": 2,
    "explanation": "Evidence over claims."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### An inventory with typed errors and Codable

Build an Inventory value type that receives and ships stock, throws a typed error for bad quantities, unknown items and insufficient stock, reports sorted lines and round-trips through JSON with Codable. It runs as an ordinary Swift script on your machine (swift main.swift); no UI is involved.

**Starter code**

```swift
import Foundation

// A tiny inventory domain: value types, enums with associated values, Result-style errors and Codable.
enum InventoryError: Error, Equatable {
    case invalidQuantity(Int)
    case unknownItem(String)
    case insufficientStock(item: String, have: Int, want: Int)
}

struct Item: Codable, Equatable {
    let sku: String
    var quantity: Int
}

// TODO: struct Inventory with private(set) var items: [String: Item]
//   mutating receive(_ sku:, _ quantity:) throws   quantity must be > 0 (else invalidQuantity); adds stock, creating the item if needed
//   mutating ship(_ sku:, _ quantity:) throws      quantity > 0, item must exist (unknownItem), enough stock (insufficientStock(item:have:want:))
//   var report: [String]                           "sku: quantity" lines sorted by sku

var inv = Inventory()
let steps: [(String, () throws -> Void)] = [
    ("receive apple 10", { try inv.receive("apple", 10) }),
    ("receive pear 4", { try inv.receive("pear", 4) }),
    ("ship apple 3", { try inv.ship("apple", 3) }),
    ("ship pear 9", { try inv.ship("pear", 9) }),
    ("ship kiwi 1", { try inv.ship("kiwi", 1) }),
    ("receive fig 0", { try inv.receive("fig", 0) }),
]
for (label, step) in steps {
    do { try step(); print("ok   \(label)") }
    catch let e as InventoryError { print("fail \(label): \(e)") }
}
print(inv.report)

let encoder = JSONEncoder()
encoder.outputFormatting = .sortedKeys
let data = try encoder.encode(inv.items.values.sorted { $0.sku < $1.sku })
let json = String(data: data, encoding: .utf8)!
print(json)
let back = try JSONDecoder().decode([Item].self, from: data)
print(back == inv.items.values.sorted { $0.sku < $1.sku })
```

**Expected output**

```text
ok   receive apple 10
ok   receive pear 4
ok   ship apple 3
fail ship pear 9: insufficientStock(item: "pear", have: 4, want: 9)
fail ship kiwi 1: unknownItem("kiwi")
fail receive fig 0: invalidQuantity(0)
["apple: 7", "pear: 4"]
[{"quantity":7,"sku":"apple"},{"quantity":4,"sku":"pear"}]
true
```

<details><summary>Show solution and explanation</summary>

<pre><code>import Foundation

// A tiny inventory domain: value types, enums with associated values, Result-style errors and Codable.
enum InventoryError: Error, Equatable {
    case invalidQuantity(Int)
    case unknownItem(String)
    case insufficientStock(item: String, have: Int, want: Int)
}

struct Item: Codable, Equatable {
    let sku: String
    var quantity: Int
}

struct Inventory {
    private(set) var items: [String: Item] = [:]

    mutating func receive(_ sku: String, _ quantity: Int) throws {
        guard quantity &gt; 0 else { throw InventoryError.invalidQuantity(quantity) }
        items[sku, default: Item(sku: sku, quantity: 0)].quantity += quantity
    }

    mutating func ship(_ sku: String, _ quantity: Int) throws {
        guard quantity &gt; 0 else { throw InventoryError.invalidQuantity(quantity) }
        guard let item = items[sku] else { throw InventoryError.unknownItem(sku) }
        guard item.quantity &gt;= quantity else {
            throw InventoryError.insufficientStock(item: sku, have: item.quantity, want: quantity)
        }
        items[sku]?.quantity -= quantity
    }

    var report: [String] {
        items.values.sorted { $0.sku &lt; $1.sku }.map { &quot;\($0.sku): \($0.quantity)&quot; }
    }
}

var inv = Inventory()
let steps: [(String, () throws -&gt; Void)] = [
    (&quot;receive apple 10&quot;, { try inv.receive(&quot;apple&quot;, 10) }),
    (&quot;receive pear 4&quot;, { try inv.receive(&quot;pear&quot;, 4) }),
    (&quot;ship apple 3&quot;, { try inv.ship(&quot;apple&quot;, 3) }),
    (&quot;ship pear 9&quot;, { try inv.ship(&quot;pear&quot;, 9) }),
    (&quot;ship kiwi 1&quot;, { try inv.ship(&quot;kiwi&quot;, 1) }),
    (&quot;receive fig 0&quot;, { try inv.receive(&quot;fig&quot;, 0) }),
]
for (label, step) in steps {
    do { try step(); print(&quot;ok   \(label)&quot;) }
    catch let e as InventoryError { print(&quot;fail \(label): \(e)&quot;) }
}
print(inv.report)

let encoder = JSONEncoder()
encoder.outputFormatting = .sortedKeys
let data = try encoder.encode(inv.items.values.sorted { $0.sku &lt; $1.sku })
let json = String(data: data, encoding: .utf8)!
print(json)
let back = try JSONDecoder().decode([Item].self, from: data)
print(back == inv.items.values.sorted { $0.sku &lt; $1.sku })
</code></pre>

<p>The guard statements validate in order, an enum with associated values carries the details, items[sku, default:] creates entries on demand and private(set) protects the invariant. Verified by running it with Swift 6.4 on macOS.</p>
</details>

### Parallel work with a task group, an actor cache and cancellation

Square a list of numbers concurrently with a task group, share an actor-protected cache between the child tasks, show that a long task stops when cancelled and combine two async let results. Timings are tiny sleeps so it finishes in under a second; only deterministic facts are printed.

**Starter code**

```swift
// Structured concurrency: parallel work with a task group, an actor-protected cache and cancellation.
// TODO: actor ResultCache with value(for:compute:) that returns a cached value or awaits compute(n) and stores it,
//       counting hits and misses (private(set))
// Note: an actor method that awaits inside can be re-entered, so two concurrent lookups of the same key may both miss.

func slowSquare(_ n: Int) async -> Int {
    try? await Task.sleep(nanoseconds: 20_000_000)
    return n * n
}

let cache = ResultCache()
let inputs = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

let squares = await withTaskGroup(of: (Int, Int).self) { group -> [Int: Int] in
    for n in inputs {
        group.addTask { (n, await cache.value(for: n, compute: slowSquare)) }
    }
    var out: [Int: Int] = [:]
    for await (n, sq) in group { out[n] = sq }
    return out
}
print(squares.keys.sorted().map { "\($0)->\(squares[$0]!)" }.joined(separator: " "))
let total = await cache.hits + cache.misses
print("lookups:", total, "unique results:", squares.count)

// cancellation: a long task observes Task.isCancelled and stops early
let worker = Task { () -> Int in
    var done = 0
    while !Task.isCancelled && done < 1_000_000 {
        done += 1
        if done % 1000 == 0 { await Task.yield() }
    }
    return done
}
worker.cancel()
let finished = await worker.value
print("cancelled early:", finished < 1_000_000)

// async let: two independent computations awaited together
async let a = slowSquare(7)
async let b = slowSquare(8)
print(await a + b)
```

**Expected output**

```text
1->1 2->4 3->9 4->16 5->25 6->36 9->81
lookups: 10 unique results: 7
cancelled early: true
113
```

<details><summary>Show solution and explanation</summary>

<pre><code>// Structured concurrency: parallel work with a task group, an actor-protected cache and cancellation.
actor ResultCache {
    private var store: [Int: Int] = [:]
    private(set) var hits = 0
    private(set) var misses = 0

    func value(for n: Int, compute: @Sendable (Int) async -&gt; Int) async -&gt; Int {
        if let v = store[n] { hits += 1; return v }
        misses += 1
        let v = await compute(n)
        store[n] = v
        return v
    }
}

func slowSquare(_ n: Int) async -&gt; Int {
    try? await Task.sleep(nanoseconds: 20_000_000)
    return n * n
}

let cache = ResultCache()
let inputs = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

let squares = await withTaskGroup(of: (Int, Int).self) { group -&gt; [Int: Int] in
    for n in inputs {
        group.addTask { (n, await cache.value(for: n, compute: slowSquare)) }
    }
    var out: [Int: Int] = [:]
    for await (n, sq) in group { out[n] = sq }
    return out
}
print(squares.keys.sorted().map { &quot;\($0)-&gt;\(squares[$0]!)&quot; }.joined(separator: &quot; &quot;))
let total = await cache.hits + cache.misses
print(&quot;lookups:&quot;, total, &quot;unique results:&quot;, squares.count)

// cancellation: a long task observes Task.isCancelled and stops early
let worker = Task { () -&gt; Int in
    var done = 0
    while !Task.isCancelled &amp;&amp; done &lt; 1_000_000 {
        done += 1
        if done % 1000 == 0 { await Task.yield() }
    }
    return done
}
worker.cancel()
let finished = await worker.value
print(&quot;cancelled early:&quot;, finished &lt; 1_000_000)

// async let: two independent computations awaited together
async let a = slowSquare(7)
async let b = slowSquare(8)
print(await a + b)
</code></pre>

<p>Each child task awaits the actor, so results are collected safely; hits plus misses always equals the lookups, while the split between them can vary because an awaiting actor method can be re-entered. Cancellation is cooperative: the worker checks Task.isCancelled. Verified by running it with Swift 6.4 on macOS.</p>
</details>
