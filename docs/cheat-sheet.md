---
title: "Swift Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Swift cheat sheet: 31 quick-reference tables from every level of the Swift Mastery Path, each linked to its full lesson."
---

# Swift Cheat Sheet

Quick-reference tables from **31 lessons** of the free Swift Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

| Type | Example | Notes |
|------|---------|-------|
| `Int` | `let x = 5` | Default integer type, platform word size |
| `Double` | `let x = 5.0` | Default floating-point type |
| `Float` | `let x: Float = 5.0` | 32-bit, smaller/less precise than `Double` |
| `String` | `let x = "hi"` | Unicode-correct, value type |
| `Character` | `let x: Character = "A"` | A single grapheme cluster |
| `Bool` | `let x = true` | `true`/`false` only, no truthy/falsy values |
| `(T, U, ...)` | `let x = (1, "a")` | Tuple — compound, unnamed type |

### [03 · Control Flow](level-1/03-control-flow.md)

| Construct | Use when |
|-----------|----------|
| `if`/`else if`/`else` | Branching on boolean conditions |
| `switch` | Matching many discrete values, ranges, or patterns exhaustively |
| `for-in` | Iterating a known sequence, range, or collection |
| `while` | Looping while a condition holds, unknown iteration count |
| `repeat-while` | Same as `while`, but body always runs once first |
| `break` / `continue` | Exiting or skipping an iteration; add a label to target an outer loop |

### [04 · Functions](level-1/04-functions.md)

| Feature | Syntax | Notes |
|---------|--------|-------|
| Argument label | `func f(label name: Type)` | Label used at call site, `name` used inside |
| Drop label | `func f(_ name: Type)` | No label required at call site |
| Default value | `func f(x: Int = 0)` | Caller may omit the argument |
| Variadic | `func f(_ xs: Int...)` | Becomes `[Int]` inside the function |
| Multiple returns | `-> (a: Int, b: String)` | Tuple return type, optionally labeled |
| Mutate caller's variable | `func f(_ x: inout Int)` | Call with `f(&value)` |
| Function type | `(Int, Int) -> Int` | Functions can be stored, passed, returned |

### [05 · Optionals Basics](level-1/05-optionals-basics.md)

| Technique | Syntax | When to use |
|-----------|--------|--------------|
| Force unwrap | `value!` | Only when `nil` is truly impossible; crashes otherwise |
| Optional binding | `if let x = value { ... }` | Use the value only inside this block |
| Early exit | `guard let x = value else { return }` | Use the value for the rest of the function |
| Default value | `value ?? defaultValue` | One-line fallback, no branching needed |
| Optional chaining | `value?.property` | Safely reach through a chain of optionals |
| Implicitly unwrapped | `String!` | Rare; auto-unwraps, still crashes if `nil` |

### [08 · Enums](level-1/08-enums.md)

| Feature | Example | Notes |
|---------|---------|-------|
| Basic case | `case north` | No associated data |
| Raw value | `case low = 1` | One fixed value per case, shared by all instances |
| Associated value | `case success(data: String)` | Different data per instance of the case |
| `rawValue` init | `Direction(rawValue: "N")` | Returns an `Optional` — invalid input yields `nil` |
| `CaseIterable` | `Suit.allCases` | Iterate every case automatically |
| `mutating func` | Changes `self` to a different case | Requires the enum variable to be `var` |

### [09 · Closures Intro](level-1/09-closures-intro.md)

| Style | Example |
|-------|---------|
| Full syntax | `{ (a: Int, b: Int) -> Int in return a + b }` |
| Inferred types | `{ a, b in return a + b }` |
| Implicit return | `{ a, b in a + b }` |
| Shorthand args | `{ $0 + $1 }` |
| Trailing closure | `numbers.sorted { $0 < $1 }` |

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

| Tool | Use it when |
|------|-------------|
| `class` inheritance | You need shared, mutable state and a genuine "is-a" relationship, usually 1-2 levels deep |
| `protocol` | You want unrelated types (struct, class, enum) to share a contract |
| `protocol extension` | You want a default implementation that conforming types can override |
| `extension` on a concrete type | You want to add behavior to a type without touching its original definition |
| `Protocol & Protocol` | A function needs to accept "anything that can do both X and Y" |
| `final` | You want to lock a class against subclassing, or a method against overriding |

### [02 · Optionals Advanced](level-2/02-optionals-advanced.md)

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

### [04 · Closures Advanced](level-2/04-closures-advanced.md)

| Capture style | Syntax | Behavior |
|----------------|--------|----------|
| Default (by reference) | `{ ... }` | Live link to the variable; sees later mutations |
| Capture list, by value | `{ [x] in ... }` | Snapshots `x`'s value at closure creation |
| Weak capture | `{ [weak self] in ... }` | Optional reference; `nil` if the object is gone; safest for escaping closures |
| Unowned capture | `{ [unowned self] in ... }` | Non-optional reference; crashes if accessed after deallocation |
| Non-escaping param (default) | `(() -> Void)` | Must be called before the function returns |
| Escaping param | `(@escaping () -> Void)` | Can be stored and called after the function returns |

### [05 · Generics](level-2/05-generics.md)

| Concept | Syntax | Purpose |
|---------|--------|---------|
| Generic function | `func f<T>(_ x: T)` | One implementation, many concrete types |
| Type constraint | `<T: Comparable>` | Require a capability so the body can use it |
| Multiple constraints | `<T: Comparable & Hashable>` | Require more than one capability |
| `where` clause | `where C1.Element == C2.Element` | Constrain relationships between type parameters |
| Generic type | `struct Stack<Element> { ... }` | A reusable container/type, specialized per use |
| `associatedtype` | `protocol Container { associatedtype Item }` | A placeholder type inside a protocol |

### [06 · Testing with XCTest](level-2/06-testing-xctest.md)

| Assertion | Checks |
|-----------|--------|
| `XCTAssertEqual(a, b)` | `a == b` |
| `XCTAssertEqual(a, b, accuracy: e)` | `abs(a - b) <= e` (for floating-point) |
| `XCTAssertTrue(x)` / `XCTAssertFalse(x)` | Boolean condition |
| `XCTAssertNil(x)` / `XCTAssertNotNil(x)` | Optional is/isn't `nil` |
| `XCTAssertThrowsError(try f())` | Expression throws |
| `XCTAssertNoThrow(try f())` | Expression does not throw |
| `setUp()` / `tearDown()` | Runs before/after every test method |

### [07 · Working with JSON](level-2/07-json-codable.md)

| Task | Tool |
|------|------|
| Struct ↔ JSON automatically | Conform to `Codable` |
| JSON → struct | `JSONDecoder().decode(Type.self, from: data)` |
| Struct → JSON | `JSONEncoder().encode(value)` |
| snake_case JSON keys | `decoder.keyDecodingStrategy = .convertFromSnakeCase` |
| Custom/partial key renaming | `enum CodingKeys: String, CodingKey` |
| Optional/missing JSON field | Make the Swift property `Optional` |
| Pretty-printed output | `encoder.outputFormatting = .prettyPrinted` |

### [08 · Swift Package Manager](level-2/08-swift-package-manager.md)

| Command | Effect |
|---------|--------|
| `swift package init --type executable` | Scaffold a runnable package |
| `swift package init --type library` | Scaffold a library package |
| `swift build` | Compile the package |
| `swift run [target]` | Build (if needed) and run an executable target |
| `swift test` | Run all XCTest targets |
| `Package.swift` | Manifest: name, platforms, dependencies, targets |
| `Package.resolved` | Locked dependency versions — commit this file |
| `internal` (default) | Visible within the module only |
| `public` | Visible to importing modules |

## Level 3 · Advanced

### [01 · Concurrency](level-3/01-concurrency.md)

| Tool | Use it when |
|------|-------------|
| `async`/`await` | Marking and calling a function that may suspend |
| `async let` | A fixed, known number of concurrent child computations |
| `Task { }` | Starting unstructured async work from sync code |
| `withTaskGroup` | A dynamic number of concurrent child tasks |
| `actor` | Protecting mutable state shared across concurrent tasks |
| `Task.sleep(nanoseconds:)` | Non-blocking delay inside async code |

### [02 · Building Server-Side APIs (Vapor)](level-3/02-server-side-apis-vapor.md)

| Concept | API |
|---------|-----|
| Define a route | `app.get("path") { req async throws -> T in ... }` |
| Path parameter | `app.get("todos", ":id")` + `req.parameters.get("id", as: Int.self)` |
| Decode request body | `try req.content.decode(MyInput.self)` |
| Return an error | `throw Abort(.notFound)` |
| Add middleware | `app.middleware.use(MyMiddleware())` |
| Group routes | `app.grouped("api", "v1")` |

### [03 · Databases](level-3/03-databases.md)

| Task | API |
|------|-----|
| Open a database | `sqlite3_open(path, &db)` |
| Run DDL/one-off SQL | `sqlite3_exec(db, sql, nil, nil, &errMsg)` |
| Prepare a parameterized statement | `sqlite3_prepare_v2(db, sql, -1, &stmt, nil)` |
| Bind a value | `sqlite3_bind_text(stmt, index, value, -1, SQLITE_TRANSIENT)` |
| Execute an insert/update | `sqlite3_step(stmt) == SQLITE_DONE` |
| Read a query row-by-row | `while sqlite3_step(stmt) == SQLITE_ROW { ... }` |
| Release a statement | `sqlite3_finalize(stmt)` |

### [04 · Design Patterns in Swift](level-3/04-design-patterns.md)

| Pattern | Swift idiom |
|---------|-------------|
| Singleton | `static let shared = Type()` + `private init()` |
| Factory | An `enum` or function returning a protocol type |
| Observer | A protocol (`AnyObject`-constrained) + a list of subscribers |
| Strategy | A protocol (or closure) stored as a property and swapped at runtime |

### [05 · iOS Fundamentals](level-3/05-ios-fundamentals.md)

| Concept | UIKit | SwiftUI |
|---------|-------|---------|
| Entry point | `AppDelegate` + `SceneDelegate` | `@main struct MyApp: App` |
| Screen | `UIViewController` | `View` (a struct) |
| One-time setup | `viewDidLoad()` | `.onAppear` (careful: can rerun) / `init` |
| Layout | `NSLayoutConstraint` anchors | Stacks (`VStack`/`HStack`) + modifiers |
| Push a screen | `navigationController?.pushViewController` | `NavigationLink` + `navigationDestination` |

### [06 · Testing Advanced](level-3/06-testing-advanced.md)

| Need | Tool |
|------|------|
| Substitute a fake dependency | A protocol + a test-only conforming type |
| Assert an interaction happened | A counter/flag property on the mock |
| Test an `async` function | `func testX() async throws { ... }` |
| Wait for a completion handler | `XCTestExpectation` + `wait(for:timeout:)` |
| Catch performance regressions | `measure { ... }` |

### [07 · Memory Management](level-3/07-memory-management.md)

| Tool | Nilable? | Use when |
|------|----------|----------|
| Strong (default) | No | The normal case; the owner should keep the referenced object alive |
| `weak` | Yes, auto-nils | A back-reference where the referenced object might be deallocated first |
| `unowned` | No, non-optional | A back-reference guaranteed to outlive the referencer |
| `[weak self]` in a closure | Yes | A closure stored by `self` that also references `self` |

### [08 · Performance &amp; Profiling](level-3/08-performance-profiling.md)

| Symptom | Likely fix |
|---------|------------|
| Slow loop appending to an array of known final size | `array.reserveCapacity(n)` |
| Slow loop building a big string | Collect pieces in an array, `joined()` once |
| Repeated `.contains(...)` checks on a large collection | Convert to `Set` once, reuse it |
| Need to find an unknown hot spot | Instruments' Time Profiler (needs full Xcode) |
| Suspected memory leak/growth | Instruments' Allocations template (needs full Xcode) |

### [09 · Protocol-Oriented Programming](level-3/09-protocol-oriented-programming.md)

| Feature | Purpose |
|---------|---------|
| `extension P { ... }` | Default implementation for a protocol requirement |
| `associatedtype Item` | A protocol-level generic placeholder |
| `A & B` | Require conformance to multiple protocols at once |
| `extension P where Self: X` / `where Element: Y` | Conditional extension members |
| `extension ExistingType: NewProtocol {}` | Retroactive conformance |

## Level 4 · Master

### [01 · Advanced Concurrency](level-4/01-advanced-concurrency.md)

| Tool | Use it for |
|------|------------|
| `AsyncStream` | Bridging a callback/delegate source into `for await` |
| Custom `AsyncSequence` | A computed sequence of values produced asynchronously |
| `Task.isCancelled` | Checking cancellation cooperatively inside a loop |
| `Task.sleep` (thrown, not `try?`ed) | Propagating cancellation automatically during a wait |
| `withThrowingTaskGroup` | Fanning out children where any failure should abort the group |

### [02 · Production Server-Side Swift](level-4/02-production-server-side-swift.md)

| Concern | Tool |
|---------|------|
| Structured logs | `swift-log`'s `Logger` + `LoggingSystem.bootstrap` |
| Environment-driven config | `ProcessInfo.processInfo.environment[...]` with defaults |
| Liveness/readiness signal | A `HealthCheck` type exposing a small enum status |
| Clean shutdown | `DispatchSource.makeSignalSource` for `SIGTERM`/`SIGINT` |

### [03 · SwiftUI Fundamentals](level-4/03-swiftui-fundamentals.md)

| Need | Tool |
|------|------|
| Local, view-owned mutable state | `@State private var` |
| Pass mutable access to a child | `@Binding` + `$` projection |
| Shared, reference-type state | `@Observable` class |
| Style/layout a view | Chained modifiers (order matters) |
| Render a dynamic collection | `List`/`ForEach` + `Identifiable` |

### [04 · Security Best Practices](level-4/04-security-best-practices.md)

| Need | `CryptoKit` API |
|------|-----------------|
| Data integrity fingerprint | `SHA256.hash(data:)` |
| Verify a message + shared secret | `HMAC<SHA256>.authenticationCode` / `.isValidAuthenticationCode` |
| Encrypt + authenticate data | `AES.GCM.seal` / `.open` |
| Compare secrets safely | `HMAC.isValidAuthenticationCode`, or a manual constant-time XOR loop |
| Store a secret on-device | Keychain (never `UserDefaults`) |

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

| Need | Tool |
|------|------|
| Fresh fixture per test | `override func setUp()` / `tearDown()` |
| Run tests concurrently | `swift test --parallel` / `-parallel-testing-enabled YES` |
| Group tests into named suites | `.xctestplan` test plans |
| CI on every push/PR | GitHub Actions workflow, `runs-on: macos-14` |
| Catch untested new code | `--enable-code-coverage` + `llvm-cov` |

### [06 · Deployment (Docker)](level-4/06-deployment-docker.md)

| Task | Command/directive |
|------|--------------------|
| Compile for a small runtime footprint | `swift build -c release --static-swift-stdlib` |
| Keep only the compiled binary | Multi-stage `FROM ... AS build` + `COPY --from=build` |
| Build an image | `docker build -t name:tag .` |
| Run with env-driven config | `docker run -p HOST:CONTAINER -e KEY=VALUE image:tag` |
| Exclude files from the build context | `.dockerignore` |
| Let the orchestrator poll liveness | `HEALTHCHECK` directive |

### [07 · Performance at Scale](level-4/07-performance-at-scale.md)

| Technique | Helps when | Doesn't help when |
|-----------|-----------|--------------------|
| Memoization | Overlapping/repeated sub-computations (recursion, shared queries) | Every input is unique — nothing to reuse |
| Batching | Per-call overhead is real (network, DB, actor hops) | Pure in-memory CPU work with no per-call cost |
| `withTaskGroup` parallelism | Chunks are independent and individually substantial | Chunks are tiny, or depend on each other's results |

### [08 · Interop with C/Objective-C](level-4/08-interop-c-objc.md)

| Need | Approach |
|------|----------|
| Call a C function | Just call it — the Clang importer exposes it directly |
| Bridge a C string | `String(cString:)`, and `free()` if it was heap-allocated |
| Pass a Swift array to a C-style API | `array.withUnsafeBufferPointer { ... }` |
| Enable KVO/KVC on a property | `@objc dynamic var` on an `NSObject` subclass |
| Bridge `String` ↔ `NSString` | `as String` / `as NSString`, transparently |

### [09 · App Architecture Patterns](level-4/09-app-architecture-patterns.md)

| Layer | Responsibility | Depends on |
|-------|----------------|------------|
| Model | Plain data | Nothing |
| Service | I/O, behind a protocol | Nothing app-specific |
| View model | Presentation state + transformation | A service protocol |
| View | Rendering + user input | A view model |
| Coordinator/Router | Navigation decisions | Route values, not views directly |

---

Go deeper with the full [Swift Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
