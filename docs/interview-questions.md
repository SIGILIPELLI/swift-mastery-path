---
title: "Swift Interview Questions and Answers (179 Questions)"
description: "179 Swift interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Swift Mastery Path."
---

# Swift Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Swift Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** Which tool manages Swift packages and builds from the command line?

**Answer:** Swift Package Manager (swift build, swift run, swift test)

SwiftPM ships with the toolchain.

**Q2.** What is the Swift REPL or playground used for?

**Answer:** Trying code interactively without creating a project

Great for learning.

**Q3.** Which platforms can run Swift today?

**Answer:** macOS, iOS and other Apple platforms, Linux and Windows

Server-side Swift runs on Linux.

**Q4.** What does swift run do in a package?

**Answer:** Builds and runs the executable target

swift test runs tests.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```swift
let name = "Ana"
var age = 30
age += 1
let pi: Double = 3.14
let n = Int(pi)
let s = "\(name) is \(age), n=\(n)"
print(s)
print(type(of: age), type(of: pi), s.count)
```

**Answer:** Ana is 31, n=3
Int Double 14

let is constant, var is mutable; Int(Double) truncates; interpolation uses \(...). Verified by compiling and running it.

**Q6.** What does type inference do?

**Answer:** The compiler deduces a variable's type from its initial value

Annotate when needed.

**Q7.** What happens when you assign a value of the wrong type?

**Answer:** A compile-time error: Swift is statically and strongly typed

No implicit conversions between numeric types.

**Q8.** What is a tuple?

**Answer:** A fixed group of values, possibly of different types, such as (Int, String)

Access by index or label.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```swift
let score = 78
let grade: String
switch score {
case 90...: grade = "A"
case 70..<90: grade = "B"
default: grade = "C"
}
var total = 0
for i in 1...4 where i % 2 == 0 { total += i }
var n = 0
while n < 3 { n += 1 }
repeat { n -= 1 } while n > 5
print(grade, total, n)
```

**Answer:** B 6 2

Ranges in switch, a where clause on for, and repeat-while running at least once. Verified by compiling and running it.

**Q10.** What must a Swift switch statement be?

**Answer:** Exhaustive: every possible value must be handled

Use default when needed.

**Q11.** Does a Swift switch case fall through by default?

**Answer:** No; each case exits the switch unless you use fallthrough

Unlike C.

**Q12.** What does guard do?

**Answer:** Exits early when a condition fails, keeping the happy path unindented

The else block must leave the scope.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```swift
func area(w: Double, h: Double = 2) -> Double { w * h }
func greet(_ name: String, from city: String) -> String { "Hi \(name) from \(city)" }
func sum(_ nums: Int...) -> Int { nums.reduce(0, +) }
func swapIt(_ a: inout Int, _ b: inout Int) { let t = a; a = b; b = t }
var x = 1, y = 2
swapIt(&x, &y)
print(area(w: 3), greet("Bo", from: "Paris"), sum(1, 2, 3), x, y)
```

**Answer:** 6.0 Hi Bo from Paris 6 2 1

Argument labels, defaults, variadics and inout. Verified by compiling and running it.

**Q14.** What are argument labels for?

**Answer:** Make call sites read like sentences (and can be omitted with _)

external versus internal names.

**Q15.** What does a function with a single expression body do?

**Answer:** Implicitly returns that expression

No return keyword needed.

**Q16.** What does inout allow?

**Answer:** A function to modify the caller's variable (passed with &amp;)

Copy-in copy-out semantics.

### [05 · Optionals Basics](level-1/05-optionals-basics.md)

**Q17.** What does this print?

```swift
var maybe: String? = "hi"
print(maybe ?? "none", maybe?.count ?? 0)
maybe = nil
print(maybe ?? "none", maybe?.count ?? 0)
if let m = maybe { print(m) } else { print("nil") }
let n: Int? = Int("42")
let bad: Int? = Int("x")
print(n!, bad == nil)
```

**Answer:** hi 2
none 0
nil
42 true

?? supplies defaults; optional chaining returns nil; failable initialisers return optionals. Verified by compiling and running it.

**Q18.** What does the force-unwrap operator ! do on nil?

**Answer:** Crashes at runtime

Prefer if let, guard let or ??.

**Q19.** What does if let do?

**Answer:** Unwraps an optional into a constant when it has a value

Shorthand: if let x { ... } in modern Swift.

**Q20.** Why do optionals exist?

**Answer:** They make the possible absence of a value explicit in the type system

No null pointer surprises.

### [06 · Collections](level-1/06-collections.md)

**Q21.** What does this print?

```swift
var a = [3, 1, 2]
a.append(4)
a.sort()
let d = ["x": 1, "y": 2]
let s: Set = [1, 2, 2, 3]
print(a, a.map { $0 * 2 }.filter { $0 > 4 }, d["z"] as Any, d["x", default: 0], s.count)
print(a.reduce(0, +), a.firstIndex(of: 3) ?? -1, Array(a[1...2]))
```

**Answer:** [1, 2, 3, 4] [6, 8] nil 1 3
10 2 [2, 3]

Dictionary subscripts return optionals; sets drop duplicates. Verified by compiling and running it.

**Q22.** Are Swift arrays value or reference types?

**Answer:** Value types with copy-on-write

Copies are cheap until mutated.

**Q23.** What does let do for an array?

**Answer:** Makes it immutable (no append or element changes)

var allows mutation.

**Q24.** Which collection has unique unordered elements?

**Answer:** Set

Elements must be Hashable.

### [07 · Structs &amp; Classes Basics](level-1/07-structs-classes.md)

**Q25.** What does this print?

```swift
struct P { var x: Int }
class C { var x: Int; init(_ x: Int) { self.x = x } }
var p1 = P(x: 1); var p2 = p1; p2.x = 9
let c1 = C(1); let c2 = c1; c2.x = 9
print(p1.x, p2.x, c1.x, c2.x, c1 === c2)
```

**Answer:** 1 9 9 9 true

Structs are copied; classes share one instance. Verified by compiling and running it.

**Q26.** When should you prefer a struct?

**Answer:** For simple data values that should be copied, which is the Swift default

Use classes for identity or inheritance.

**Q27.** What does a mutating method do?

**Answer:** Allows a struct method to modify the struct's properties

Required for value types.

**Q28.** What is a computed property?

**Answer:** A property that calculates its value on access instead of storing it

Has get and optionally set.

### [08 · Enums](level-1/08-enums.md)

**Q29.** What does this print?

```swift
enum Shape { case circle(r: Double), rect(w: Double, h: Double), point }
func area(_ s: Shape) -> Double {
  switch s { case .circle(let r): return 3 * r * r
  case .rect(let w, let h): return w * h
  case .point: return 0 }
}
enum Level: Int, CaseIterable { case low = 1, mid, high }
print(area(.circle(r: 2)), area(.rect(w: 2, h: 3)), area(.point))
print(Level.allCases.map(\.rawValue), Level(rawValue: 2) as Any, Level(rawValue: 9) as Any)
```

**Answer:** 12.0 6.0 0.0
[1, 2, 3] Optional(main.Level.mid) nil

Associated values carry data; raw values continue counting; rawValue init is failable. Verified by compiling and running it.

**Q30.** What are associated values?

**Answer:** Data attached to enum cases, different per case

Model alternatives precisely.

**Q31.** What does CaseIterable provide?

**Answer:** allCases, a collection of every case

Synthesised automatically.

**Q32.** Why are enums good for state?

**Answer:** Illegal states can be made unrepresentable

Switch must be exhaustive.

### [09 · Closures Intro](level-1/09-closures-intro.md)

**Q33.** What does this print?

```swift
let nums = [1, 2, 3, 4]
let doubled = nums.map { $0 * 2 }
let sorted = nums.sorted { $0 > $1 }
var counter = 0
let inc = { counter += 1; return counter }
_ = inc(); _ = inc()
func makeAdder(_ n: Int) -> (Int) -> Int { { $0 + n } }
print(doubled, sorted, counter, makeAdder(5)(3))
```

**Answer:** [2, 4, 6, 8] [4, 3, 2, 1] 2 8

Closures capture variables by reference. Verified by compiling and running it.

**Q34.** What is trailing closure syntax?

**Answer:** Writing the final closure argument after the parentheses

names.sorted { $0 &lt; $1 }.

**Q35.** What do $0 and $1 mean?

**Answer:** Shorthand for the first and second closure arguments

Use names when clearer.

**Q36.** What does @escaping mean?

**Answer:** The closure may be called after the function returns

Needed for stored or async closures.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-cli.md)

**Q37.** What does this todo model print?

```swift
struct Todo { let id: Int; var title: String; var done = false }
var todos: [Todo] = []
func add(_ t: String) { todos.append(Todo(id: todos.count + 1, title: t)) }
func complete(_ id: Int) -> Bool { guard let i = todos.firstIndex(where: { $0.id == id }) else { return false }; todos[i].done = true; return true }
add("milk"); add("report")
print(complete(2), complete(9))
print(todos.map { "\($0.done ? "x" : " ") \($0.title)" })
```

**Answer:** true false
["  milk", "x report"]

firstIndex(where:) with guard handles a missing id. Verified by compiling and running it.

**Q38.** How would you persist todos between runs of the CLI?

**Answer:** Encode them to JSON with Codable and write to a file

Handle missing or corrupt files.

**Q39.** How do you read command-line arguments?

**Answer:** CommandLine.arguments (or swift-argument-parser)

The first element is the program path.

**Q40.** Why keep logic in functions separate from main?

**Answer:** So it can be unit tested

Thin entry point.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```swift
class Animal { var name: String; init(_ n: String) { name = n }; func speak() -> String { "..." } }
class Dog: Animal { override func speak() -> String { "\(name): woof" } }
protocol Describable { var summary: String { get } }
extension Animal: Describable { var summary: String { "Animal \(name)" } }
let pets: [Animal] = [Animal("Generic"), Dog("Rex")]
print(pets.map { $0.speak() }, pets[1].summary, pets[1] is Dog)
```

**Answer:** ["...", "Rex: woof"] Animal Rex true

Dynamic dispatch picks the subclass override; extensions add protocol conformance. Verified by compiling and running it.

**Q42.** What does final do on a class?

**Answer:** Prevents subclassing (or overriding)

Enables optimisations.

**Q43.** What is the difference between a designated and a convenience initialiser?

**Answer:** Designated initialisers fully initialise; convenience ones delegate to them

Two-phase initialisation.

**Q44.** What does access control (private, internal, public) do?

**Answer:** Limits where declarations can be used

fileprivate and open also exist.

### [02 · Optionals Advanced](level-2/02-optionals-advanced.md)

**Q45.** What does this print?

```swift
let a: Int? = 5, b: Int? = nil
print(a.map { $0 * 2 } as Any, b.map { $0 * 2 } as Any)
print(a.flatMap { $0 > 3 ? $0 : nil } as Any)
let words = ["1", "x", "3"]
print(words.compactMap { Int($0) })
struct U { var addr: Addr?; struct Addr { var city: String? } }
let u = U(addr: .init(city: "Oslo"))
print(u.addr?.city ?? "none", U().addr?.city ?? "none")
```

**Answer:** Optional(10) nil
Optional(5)
[1, 3]
Oslo none

map and flatMap transform optionals; compactMap drops nils. Verified by compiling and running it.

**Q46.** What does the nil-coalescing operator ?? do?

**Answer:** Provides a default when the left side is nil

Chain with other defaults.

**Q47.** When is an implicitly unwrapped optional (Int!) acceptable?

**Answer:** Rarely, for values set right after initialisation such as outlets

Prefer regular optionals.

**Q48.** What does optional chaining return?

**Answer:** An optional result, nil if any link in the chain is nil

a?.b?.c.

### [03 · Error Handling](level-2/03-error-handling.md)

**Q49.** What does this print?

```swift
enum E: Error { case empty, tooShort(Int) }
func validate(_ s: String) throws -> String {
  guard !s.isEmpty else { throw E.empty }
  guard s.count >= 3 else { throw E.tooShort(s.count) }
  return s.uppercased()
}
for s in ["abc", "", "ab"] {
  do { print(try validate(s)) }
  catch E.empty { print("empty") }
  catch E.tooShort(let n) { print("short \(n)") }
}
print((try? validate("ab")) as Any, (try? validate("abcd")) as Any)
```

**Answer:** ABC
empty
short 2
nil Optional("ABCD")

try? converts errors to nil; catch patterns match specific errors. Verified by compiling and running it.

**Q50.** What does try! do when the call throws?

**Answer:** Crashes at runtime

Avoid except for truly impossible failures.

**Q51.** What does rethrows mean?

**Answer:** A function throws only if a closure argument throws

map is rethrows.

**Q52.** What does defer do?

**Answer:** Runs code when the scope exits, however it exits

Good for cleanup.

### [04 · Closures Advanced](level-2/04-closures-advanced.md)

**Q53.** What does this print?

```swift
func makeCounter() -> () -> Int { var c = 0; return { c += 1; return c } }
let a = makeCounter(), b = makeCounter()
_ = a(); _ = a()
print(a(), b())
class Owner { var name = "o"; var cb: (() -> String)?; func setup() { cb = { [weak self] in self?.name ?? "gone" } } }
var o: Owner? = Owner(); o!.setup()
let cb = o!.cb!
print(cb())
o = nil
print(cb())
```

**Answer:** 3 1
o
gone

Each closure captures its own variable; [weak self] avoids retain cycles and becomes nil. Verified by compiling and running it.

**Q54.** What causes a retain cycle with closures?

**Answer:** A closure that strongly captures self while self stores the closure

Use [weak self] or [unowned self].

**Q55.** What is the difference between weak and unowned?

**Answer:** weak is optional and becomes nil; unowned assumes the object outlives the closure and crashes if not

Choose carefully.

**Q56.** What does a closure capture list [x] do?

**Answer:** Captures x by value at the time the closure is created

Without it, variables are captured by reference.

### [05 · Generics](level-2/05-generics.md)

**Q57.** What does this print?

```swift
func largest<T: Comparable>(_ xs: [T]) -> T? { xs.max() }
struct Stack<Element> { private var items: [Element] = []; mutating func push(_ e: Element) { items.append(e) }; mutating func pop() -> Element? { items.popLast() }; var count: Int { items.count } }
var s = Stack<String>(); s.push("a"); s.push("b")
print(largest([3, 9, 2]) as Any, largest(["b", "a"]) as Any, largest([Int]()) as Any)
print(s.pop() as Any, s.count)
extension Array where Element: Numeric { func total() -> Element { reduce(0, +) } }
print([1.5, 2.5].total())
```

**Answer:** Optional(9) Optional("b") nil
Optional("b") 1
4.0

Generic functions use constraints; generic types hold any Element; constrained extensions add methods. Verified by compiling and running it.

**Q58.** What does &lt;T: Comparable&gt; mean?

**Answer:** T must conform to Comparable

Constraints enable operations.

**Q59.** What does an associated type do in a protocol?

**Answer:** Declares a placeholder type that conforming types specify

Sequence has Element.

**Q60.** What does some Protocol (opaque type) mean?

**Answer:** A specific but hidden concrete type conforming to the protocol

any Protocol is an existential.

### [06 · Testing with XCTest](level-2/06-testing-xctest.md)

**Q61.** Which framework does modern Swift use for tests?

**Answer:** Swift Testing (with XCTest still supported)

@Test and #expect macros.

**Q62.** What does XCTAssertEqual check?

**Answer:** That two values are equal, failing the test otherwise

Provide messages for clarity.

**Q63.** What makes a good unit test?

**Answer:** Fast, isolated and deterministic

Avoid real network and clocks.

**Q64.** How do you test code that depends on the network?

**Answer:** Inject a protocol-based client and use a fake in tests

Dependency injection.

### [07 · Working with JSON](level-2/07-json-codable.md)

**Q65.** What does this Codable round trip print?

```swift
import Foundation
struct User: Codable { let id: Int; var name: String; var tags: [String] = []; enum CodingKeys: String, CodingKey { case id, name, tags = "labels" } }
let json = #"{"id":1,"name":"Ana","labels":["a","b"]}"#.data(using: .utf8)!
let u = try JSONDecoder().decode(User.self, from: json)
print(u.id, u.name, u.tags)
let enc = JSONEncoder(); enc.outputFormatting = .sortedKeys
print(String(data: try enc.encode(u), encoding: .utf8)!)
print((try? JSONDecoder().decode(User.self, from: Data("{}".utf8))) == nil)
```

**Answer:** 1 Ana ["a", "b"]
{"id":1,"labels":["a","b"],"name":"Ana"}
true

CodingKeys rename fields; missing required keys make decoding throw. Verified by compiling and running it.

**Q66.** What does Codable combine?

**Answer:** Encodable and Decodable

Compiler synthesises it.

**Q67.** How do you handle date formats in JSON?

**Answer:** Set dateDecodingStrategy (for example .iso8601)

Be explicit.

**Q68.** What should you do with untrusted JSON?

**Answer:** Decode into typed models and handle errors, never force-try

Validate semantics too.

### [08 · Swift Package Manager](level-2/08-swift-package-manager.md)

**Q69.** What does Package.swift define?

**Answer:** The package's name, products, dependencies and targets

Written in Swift.

**Q70.** How do you add a dependency?

**Answer:** In Package.swift dependencies with a URL and version requirement

swift package resolve fetches it.

**Q71.** What does Package.resolved record?

**Answer:** The exact versions used (commit for reproducible builds)

Commit it for apps.

**Q72.** What are targets and products?

**Answer:** Targets are modules you build; products are what other packages can use (libraries, executables)

Test targets depend on library targets.

### [09 · Property Wrappers Intro](level-2/09-property-wrappers.md)

**Q73.** What does this property wrapper print?

```swift
@propertyWrapper struct Clamped { private var v: Int; let r: ClosedRange<Int>; var wrappedValue: Int { get { v } set { v = min(max(newValue, r.lowerBound), r.upperBound) } }; init(wrappedValue: Int, _ r: ClosedRange<Int>) { self.r = r; self.v = min(max(wrappedValue, r.lowerBound), r.upperBound) } }
struct Vol { @Clamped(0...10) var level = 5 }
var v = Vol()
v.level = 42
print(v.level)
v.level = -3
print(v.level)
```

**Answer:** 10
0

The wrapper intercepts reads and writes to enforce a range. Verified by compiling and running it.

**Q74.** What does @State do in SwiftUI?

**Answer:** Stores local view state and re-renders the view when it changes

A property wrapper.

**Q75.** What do property wrappers reduce?

**Answer:** Repeated getter/setter boilerplate

Reusable behaviour.

**Q76.** What is projectedValue ($name)?

**Answer:** An extra value a wrapper exposes, like a Binding with @State

Accessed with $.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** What does the weather CLI project combine?

**Answer:** Networking with URLSession, Codable, error handling and argument parsing

Level 2 capstone.

**Q78.** How do you make an HTTP request in modern Swift?

**Answer:** try await URLSession.shared.data(from: url)

Check the HTTP status.

**Q79.** How do you test the API layer without the network?

**Answer:** Protocol-based client plus a fake returning sample JSON

Deterministic tests.

**Q80.** Where should the API key come from?

**Answer:** Environment or configuration, not source code

Never commit keys.

## Level 3 · Advanced

### [01 · Concurrency](level-3/01-concurrency.md)

**Q81.** What does this structured-concurrency example print?

```swift
func square(_ n: Int) async -> Int { n * n }
actor Counter { var n = 0; func inc() -> Int { n += 1; return n } }
let c = Counter()
let results = await withTaskGroup(of: Int.self) { g in
  for i in 1...4 { g.addTask { await square(i) } }
  var all: [Int] = []
  for await r in g { all.append(r) }
  return all.sorted()
}
print(results)
await c.inc(); await c.inc()
print(await c.inc())
async let x = square(3)
async let y = square(4)
print(await x + y)
```

**Answer:** [1, 4, 9, 16]
3
25

Task groups run children concurrently; actors serialise access; async let starts parallel work. Verified by compiling and running it.

**Q82.** What does await mean?

**Answer:** A potential suspension point where other work can run

The thread is not blocked.

**Q83.** What is an actor?

**Answer:** A reference type that protects its mutable state by serialising access

Access across actors needs await.

**Q84.** What does Sendable indicate?

**Answer:** A type is safe to pass between concurrency domains

Swift 6 enforces it.

### [02 · Building Server-Side APIs (Vapor)](level-3/02-server-side-apis-vapor.md)

**Q85.** Which framework is the leading server-side Swift web framework?

**Answer:** Vapor (and Hummingbird)

Built on SwiftNIO.

**Q86.** What does SwiftNIO provide?

**Answer:** An event-driven networking foundation

Non-blocking I/O.

**Q87.** How does Vapor map routes?

**Answer:** app.get("path") { req in ... } closures returning Content

Typed routing with async handlers.

**Q88.** Why avoid blocking calls in async handlers?

**Answer:** They tie up event-loop or cooperative threads and hurt throughput

Use async APIs.

### [03 · Databases](level-3/03-databases.md)

**Q89.** What does Fluent provide in Vapor?

**Answer:** An ORM with models, migrations and query building

Supports Postgres, MySQL, SQLite.

**Q90.** Why use migrations?

**Answer:** Versioned, repeatable schema changes

Run on deploy.

**Q91.** How do you avoid SQL injection with a Swift database library?

**Answer:** Use parameter binding or the query builder, never string interpolation

Bind values.

**Q92.** What does a connection pool do?

**Answer:** Reuses a limited number of connections across requests

Size it to the database.

### [04 · Design Patterns in Swift](level-3/04-design-patterns.md)

**Q93.** What does this Strategy and Observer model print?

```swift
protocol Pricing { func price(_ base: Double) -> Double }
struct Regular: Pricing { func price(_ b: Double) -> Double { b } }
struct Sale: Pricing { let pct: Double; func price(_ b: Double) -> Double { b * (1 - pct) } }
final class Cart { var strategy: Pricing; var subs: [(Double) -> Void] = []; init(_ s: Pricing) { strategy = s }
  func checkout(_ base: Double) { let p = strategy.price(base); subs.forEach { $0(p) } } }
let cart = Cart(Regular())
var seen: [Double] = []
cart.subs.append { seen.append($0) }
cart.checkout(100)
cart.strategy = Sale(pct: 0.25)
cart.checkout(100)
print(seen)
```

**Answer:** [100.0, 75.0]

Swapping the strategy changes behaviour; closures act as observers. Verified by compiling and running it.

**Q94.** What is dependency injection in Swift?

**Answer:** Passing collaborators in (often as protocols) instead of creating them inside

Improves testability.

**Q95.** What is a downside of singletons?

**Answer:** Hidden global state that makes tests and reasoning harder

Inject instead.

**Q96.** What does the Factory pattern do?

**Answer:** Encapsulates object creation behind a function or type

Use enums or static funcs.

### [05 · iOS Fundamentals](level-3/05-ios-fundamentals.md)

**Q97.** What does SwiftUI use to describe interfaces?

**Answer:** Declarative View structs composed in a body

State drives the UI.

**Q98.** What is the iOS app lifecycle entry in SwiftUI?

**Answer:** A type marked @main conforming to App

Scenes contain windows.

**Q99.** What is Auto Layout / SwiftUI stacks for?

**Answer:** Adapting layout to different screen sizes and orientations

HStack, VStack, ZStack.

**Q100.** Why test on devices and not only simulators?

**Answer:** Hardware, performance and sensors differ

Also check different iOS versions.

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

**Q101.** What does a UI test do?

**Answer:** Drives the app's interface like a user (XCUITest)

Slower, so use sparingly.

**Q102.** What is a test double?

**Answer:** A fake, stub or mock standing in for a real dependency

Use protocol-based fakes.

**Q103.** What is code coverage a signal of?

**Answer:** Which code the tests execute, not whether behaviour is correct

Use it to find untested areas.

**Q104.** Why keep tests deterministic?

**Answer:** Flaky tests erode trust

Control time and randomness.

### [07 · Memory Management](level-3/07-memory-management.md)

**Q105.** What does this ARC demo print?

```swift
final class Node { let name: String; weak var friend: Node?; init(_ n: String) { name = n }; deinit { print("deinit \(name)") } }
var a: Node? = Node("a")
var b: Node? = Node("b")
a?.friend = b; b?.friend = a
a = nil
print("a released")
b = nil
print("done")
```

**Answer:** deinit a
a released
deinit b
done

weak references break cycles, so both objects are released as soon as the strong references go. Verified by compiling and running it.

**Q106.** What does ARC do?

**Answer:** Automatically counts references and frees objects when the count reaches zero

No GC pauses.

**Q107.** What causes a memory leak in Swift?

**Answer:** A retain cycle where objects keep each other alive

Use weak or unowned.

**Q108.** Which tool finds leaks?

**Answer:** The Memory Graph Debugger and Instruments

Check deinit logging.

### [08 · Performance &amp; Profiling](level-3/08-performance-profiling.md)

**Q109.** What should you do first when an app is slow?

**Answer:** Profile with Instruments (Time Profiler) to find the hot path

Measure before optimising.

**Q110.** What is a common main-thread mistake in iOS?

**Answer:** Doing heavy work or blocking I/O on the main thread, freezing the UI

Move work to background tasks.

**Q111.** Why prefer structs and value semantics for performance?

**Answer:** Often allocated on the stack and no reference counting overhead

Not always, so measure.

**Q112.** What does lazy do for a property or collection?

**Answer:** Defers computation until needed

lazy var and lazy sequences.

### [09 · Protocol-Oriented Programming](level-3/09-protocol-oriented-programming.md)

**Q113.** What does this protocol-oriented example print?

```swift
protocol Named { var name: String { get } }
protocol Greeter: Named { func greet() -> String }
extension Greeter { func greet() -> String { "Hello, \(name)" } }
struct Person: Greeter { let name: String }
struct Robot: Greeter { let name: String; func greet() -> String { "BEEP \(name)" } }
let gs: [any Greeter] = [Person(name: "Ana"), Robot(name: "R2")]
print(gs.map { $0.greet() })
func loud<G: Greeter>(_ g: G) -> String { g.greet().uppercased() }
print(loud(Person(name: "bo")))
```

**Answer:** ["Hello, Ana", "BEEP R2"]
HELLO, BO

Protocol extensions supply defaults that conforming types may override. Verified by compiling and running it.

**Q114.** What is protocol-oriented programming?

**Answer:** Designing around protocols and composition with default implementations instead of class inheritance

Value types participate.

**Q115.** What is the difference between any P and some P?

**Answer:** any is a type-erased existential; some hides one fixed concrete type

Prefer some or generics when possible.

**Q116.** Why prefer composition over inheritance?

**Answer:** Flexible, avoids deep hierarchies and works with structs

Small protocols.

### [10 · Project — REST API + Database Service](level-3/10-project-rest-api-db.md)

**Q117.** What does the REST API + DB project combine?

**Answer:** Vapor routes, Fluent models, migrations, validation and tests

Level 3 capstone.

**Q118.** How do you validate request input in Vapor?

**Answer:** Validatable with rules, returning 400 with details

Never trust clients.

**Q119.** Which status for a duplicate unique key?

**Answer:** 409 Conflict

Map database errors.

**Q120.** How do you test routes?

**Answer:** XCTVapor to send in-memory requests

Fast and isolated.

## Level 4 · Master

### [01 · Advanced Concurrency](level-4/01-advanced-concurrency.md)

**Q121.** What is structured concurrency?

**Answer:** Child tasks have a defined scope and lifetime tied to their parent, so cancellation and errors propagate

Task groups and async let.

**Q122.** How does cancellation work in Swift concurrency?

**Answer:** Cooperative: tasks check Task.isCancelled or call try Task.checkCancellation()

Respect it in long loops.

**Q123.** What does @MainActor guarantee?

**Answer:** Code runs on the main thread

Use for UI updates.

**Q124.** What is a data race and how does Swift 6 help?

**Answer:** Unsynchronised concurrent access to mutable state; strict concurrency checks catch many at compile time

Sendable and actors.

### [02 · Production Server-Side Swift](level-4/02-production-server-side-swift.md)

**Q125.** What belongs in a production server-side service?

**Answer:** Structured logging, metrics, health checks, timeouts and graceful shutdown

swift-log and swift-metrics.

**Q126.** What does graceful shutdown do?

**Answer:** Finishes in-flight requests before stopping

Handle SIGTERM.

**Q127.** Why configure timeouts for outgoing requests?

**Answer:** Hung dependencies would otherwise exhaust resources

Plus retries with backoff.

**Q128.** How should secrets be provided?

**Answer:** Environment or secret store

Rotate.

### [03 · SwiftUI Fundamentals](level-4/03-swiftui-fundamentals.md)

**Q129.** What does this SwiftUI-style state model print?

```swift
struct Counter { private(set) var count = 0; mutating func inc() { count += 1 }; var label: String { "Count: \(count)" } }
var c = Counter()
var renders: [String] = []
func render() { renders.append(c.label) }
render(); c.inc(); render(); c.inc(); render()
print(renders)
```

**Answer:** ["Count: 0", "Count: 1", "Count: 2"]

SwiftUI re-evaluates body when @State changes; this model re-renders manually to show the idea (SwiftUI itself was not run). Verified by compiling and running it.

**Q130.** What does @Binding do?

**Answer:** Creates a two-way reference to state owned elsewhere

Pass with $.

**Q131.** When use @Observable (or ObservableObject)?

**Answer:** For shared model objects whose changes should refresh views

Keep views simple.

**Q132.** What does the body of a View return?

**Answer:** Some View describing the UI for the current state

Views are cheap value types.

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

**Q133.** Where should sensitive data such as tokens be stored on iOS?

**Answer:** The Keychain

Use appropriate accessibility options.

**Q134.** What does App Transport Security enforce?

**Answer:** HTTPS by default for network connections

Add exceptions only when necessary.

**Q135.** Why validate certificates and avoid disabling TLS checks?

**Answer:** Disabling them enables man-in-the-middle attacks

Pinning adds more safety.

**Q136.** What should you never log?

**Answer:** Tokens, passwords and personal data

Redact.

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

**Q137.** What does swift test do in CI?

**Answer:** Builds and runs all test targets

Run on every push.

**Q138.** Why cache SwiftPM dependencies in CI?

**Answer:** Faster builds

Key on Package.resolved.

**Q139.** What does a matrix build cover?

**Answer:** Several OS or Swift versions in one workflow

Catch portability issues.

**Q140.** What should gate merges?

**Answer:** Passing tests, lint and review

Branch protection.

### [06 · Deployment (Docker)](level-4/06-deployment-docker.md)

**Q141.** Why use a multi-stage Dockerfile for server-side Swift?

**Answer:** Build with the full toolchain and ship only the binary on a slim runtime image

Smaller attack surface.

**Q142.** Why run as non-root in the container?

**Answer:** Limits damage if compromised

Create a user.

**Q143.** What does static linking of the Swift stdlib help with?

**Answer:** Fewer runtime dependencies in the image

--static-swift-stdlib.

**Q144.** How do you configure the app per environment?

**Answer:** Environment variables read at startup

Same image everywhere.

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

**Q145.** What is a safe optimisation workflow?

**Answer:** Measure, change one thing, measure again

Instruments and benchmarks.

**Q146.** What causes excessive memory growth in long-running services?

**Answer:** Unbounded caches, retain cycles or accumulated tasks

Monitor memory.

**Q147.** Why batch database queries?

**Answer:** Reduce round trips and latency

Avoid N+1.

**Q148.** What does load testing reveal?

**Answer:** Where throughput and latency degrade under realistic traffic

Test before launch.

### [08 · Interop with C/Objective-C](level-4/08-interop-c-objc.md)

**Q149.** How does Swift call C code?

**Answer:** Through module maps or by importing C headers directly into a Swift package

System library targets.

**Q150.** What does @objc do?

**Answer:** Exposes a Swift declaration to the Objective-C runtime

Needed for selectors and interop.

**Q151.** What is an UnsafePointer?

**Answer:** A pointer without safety guarantees, needed for some C APIs

Keep unsafe code small.

**Q152.** Why wrap C APIs in safe Swift types?

**Answer:** Contain unsafety, manage lifetimes and give idiomatic APIs

Own and free resources.

### [09 · App Architecture Patterns](level-4/09-app-architecture-patterns.md)

**Q153.** What is MVVM?

**Answer:** Model-View-ViewModel: views display state exposed by a view model that wraps models and logic

Common with SwiftUI.

**Q154.** What problem does unidirectional data flow (TCA, Redux-style) solve?

**Answer:** Predictable state changes through actions and reducers

Easier testing.

**Q155.** Why separate networking, domain and UI layers?

**Answer:** Testability and replaceable components

Dependency inversion.

**Q156.** What is a Coordinator pattern for?

**Answer:** Managing navigation flow outside view controllers or views

Decouples screens.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What is the capstone's aim?

**Answer:** A production-style Swift project with architecture, tests, CI and deployment or distribution

Bring Level 4 together.

**Q158.** What proves it is production-ready?

**Answer:** CI with tests, linting, performance checks and operational readiness

Evidence.

**Q159.** What belongs in the README?

**Answer:** Setup, run, test, architecture and decisions

Handover.

**Q160.** How do you handle residual risks?

**Answer:** Document them and accept, mitigate or transfer

Risk register.

## Mixed questions across all levels

**Q161.** What does this print?

```swift
struct Box { var items: [Int] }
var a = Box(items: [1])
var b = a
b.items.append(2)
final class Ref { var items: [Int] = [1] }
let r1 = Ref(); let r2 = r1; r2.items.append(2)
print(a.items, b.items, r1.items, r1 === r2)
```

**Answer:** [1] [1, 2] [1, 2] true

Structs copy on assignment; class instances are shared references. Verified by compiling and running it.

**Q162.** What does this print?

```swift
func parse(_ s: String) -> Int? { Int(s) }
let inputs = ["4", "x", "7"]
let nums = inputs.compactMap(parse)
let total = inputs.compactMap(parse).reduce(0, +)
let firstBad = inputs.first { parse($0) == nil } ?? "none"
print(nums, total, firstBad)
let d: [String: Int] = ["a": 1]
print(d["a"] ?? -1, d["b"] ?? -1)
```

**Answer:** [4, 7] 11 x
1 -1

compactMap drops nils and ?? gives defaults for missing keys. Verified by compiling and running it.

**Q163.** What does this print?

```swift
enum Cmd { case add(Int), mul(Int), reset }
func apply(_ c: Cmd, to v: Int) -> Int {
  switch c { case .add(let n): return v + n
  case .mul(let n): return v * n
  case .reset: return 0 }
}
let result = [Cmd.add(3), .mul(4), .add(-2), .reset, .add(5)].reduce(0) { apply($1, to: $0) }
print(result)
```

**Answer:** 5

The reducer applies each command in order; reset discards everything before it. Verified by compiling and running it.

**Q164.** Why is force-unwrapping (!) discouraged outside tests and truly impossible cases?

**Answer:** It crashes at runtime when the optional is nil

Use guard let, if let or ??.

**Q165.** What does guard let do for readability?

**Answer:** Handles the failure case early and exits, so the main logic stays unindented

The else branch must leave the scope.

**Q166.** What does a Swift switch require?

**Answer:** To be exhaustive: every case handled or a default

Adding an enum case then flags every switch.

**Q167.** When do you reach for a class instead of a struct?

**Answer:** When you need shared identity, inheritance or reference semantics

Structs are the default.

**Q168.** What creates a retain cycle with closures?

**Answer:** A closure stored by an object strongly capturing that object (self)

Use [weak self] or [unowned self].

**Q169.** What does throws plus do/catch give you over returning nil?

**Answer:** Specific error information the caller can handle differently

Optionals lose the reason.

**Q170.** What does Codable synthesis give you?

**Answer:** Automatic JSON (and other format) encoding and decoding for conforming types

Custom CodingKeys rename fields.

**Q171.** What does await mark?

**Answer:** A possible suspension point where the task may yield its thread to other work

Not a blocking call.

**Q172.** What does an actor protect?

**Answer:** Its mutable state, by serialising access to it

Beware reentrancy at await points.

**Q173.** What does structured concurrency guarantee about child tasks?

**Answer:** They finish (or are cancelled) before their scope exits, and errors and cancellation propagate

Task groups and async let.

**Q174.** Where do you store tokens on iOS?

**Answer:** The Keychain, not UserDefaults

Choose an appropriate accessibility level.

**Q175.** Why prefer protocols and dependency injection for network code?

**Answer:** Tests can inject fakes and avoid real network calls

Deterministic tests.

**Q176.** What does ARC do?

**Answer:** Frees objects deterministically when their reference count reaches zero

Cycles need weak references.

**Q177.** Why should heavy work stay off the main thread in an app?

**Answer:** The main thread draws the UI, so blocking it freezes the interface

Use Task or background actors.

**Q178.** What does a multi-stage Dockerfile do for a server-side Swift service?

**Answer:** Builds with the full toolchain and ships only the binary in a slim runtime image

Smaller attack surface.

**Q179.** What proves a Swift project is production-ready?

**Answer:** CI with tests, linting, performance checks and operational readiness

Evidence over claims.

## Practical coding questions

### An inventory with typed errors and Codable

Build an Inventory value type that receives and ships stock, throws a typed error for bad quantities, unknown items and insufficient stock, reports sorted lines and round-trips through JSON with Codable. It runs as an ordinary Swift script on your machine (swift main.swift); no UI is involved.

Starter code:

```
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

Expected output:

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

??? success "Show a solution"

    ```
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

    struct Inventory {
        private(set) var items: [String: Item] = [:]

        mutating func receive(_ sku: String, _ quantity: Int) throws {
            guard quantity > 0 else { throw InventoryError.invalidQuantity(quantity) }
            items[sku, default: Item(sku: sku, quantity: 0)].quantity += quantity
        }

        mutating func ship(_ sku: String, _ quantity: Int) throws {
            guard quantity > 0 else { throw InventoryError.invalidQuantity(quantity) }
            guard let item = items[sku] else { throw InventoryError.unknownItem(sku) }
            guard item.quantity >= quantity else {
                throw InventoryError.insufficientStock(item: sku, have: item.quantity, want: quantity)
            }
            items[sku]?.quantity -= quantity
        }

        var report: [String] {
            items.values.sorted { $0.sku < $1.sku }.map { "\($0.sku): \($0.quantity)" }
        }
    }

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

    The guard statements validate in order, an enum with associated values carries the details, items[sku, default:] creates entries on demand and private(set) protects the invariant. Verified by running it with Swift 6.4 on macOS.

### Parallel work with a task group, an actor cache and cancellation

Square a list of numbers concurrently with a task group, share an actor-protected cache between the child tasks, show that a long task stops when cancelled and combine two async let results. Timings are tiny sleeps so it finishes in under a second; only deterministic facts are printed.

Starter code:

```
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

Expected output:

```text
1->1 2->4 3->9 4->16 5->25 6->36 9->81
lookups: 10 unique results: 7
cancelled early: true
113
```

??? success "Show a solution"

    ```
    // Structured concurrency: parallel work with a task group, an actor-protected cache and cancellation.
    actor ResultCache {
        private var store: [Int: Int] = [:]
        private(set) var hits = 0
        private(set) var misses = 0

        func value(for n: Int, compute: @Sendable (Int) async -> Int) async -> Int {
            if let v = store[n] { hits += 1; return v }
            misses += 1
            let v = await compute(n)
            store[n] = v
            return v
        }
    }

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

    Each child task awaits the actor, so results are collected safely; hits plus misses always equals the lookups, while the split between them can vary because an awaiting actor method can be re-entered. Cancellation is cooperative: the worker checks Task.isCancelled. Verified by running it with Swift 6.4 on macOS.

---

More free practice: the [Swift Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
