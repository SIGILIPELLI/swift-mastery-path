# 03 · Error Handling

Optionals model "a value might be missing." Swift's error handling models
something different: "this operation might fail, and I want to know *why*."
Instead of returning `nil` and losing all context, a failing operation
`throw`s a specific error value that the caller can inspect, log, or react
to differently depending on what went wrong.

## The `Error` protocol

Any type can represent an error — enums are the most common choice because
they let you enumerate every failure case explicitly:

```swift
enum ValidationError: Error {
    case tooShort(minimum: Int)
    case tooLong(maximum: Int)
    case containsInvalidCharacters
}
```

Conforming to `Error` doesn't require any methods — it's a marker protocol
that tells the compiler "this type is allowed to be thrown."

## `throws`, `try`, and `throw`

A function that can fail is marked `throws` in its signature. Inside, you
`throw` a specific error; callers must call it with `try` and handle the
possibility of failure:

```swift
func validateUsername(_ name: String) throws -> String {
    guard name.count >= 3 else {
        throw ValidationError.tooShort(minimum: 3)
    }
    guard name.count <= 20 else {
        throw ValidationError.tooLong(maximum: 20)
    }
    guard name.allSatisfy({ $0.isLetter || $0.isNumber }) else {
        throw ValidationError.containsInvalidCharacters
    }
    return name
}
```

## `do` / `try` / `catch`

`do`/`catch` is how you actually run a throwing function and respond to
failure. Multiple `catch` clauses can pattern-match specific error cases:

```swift
func register(username: String) {
    do {
        let valid = try validateUsername(username)
        print("Registered: \(valid)")
    } catch ValidationError.tooShort(let minimum) {
        print("Username must be at least \(minimum) characters")
    } catch ValidationError.tooLong(let maximum) {
        print("Username must be at most \(maximum) characters")
    } catch ValidationError.containsInvalidCharacters {
        print("Username can only contain letters and numbers")
    } catch {
        // Fallback: catches anything not matched above
        print("Unknown error: \(error)")
    }
}

register(username: "ab")           // Username must be at least 3 characters
register(username: "valid_user")   // Username can only contain letters and numbers
register(username: "validuser")    // Registered: validuser
```

The second call is a good reminder to trace through validation rules
carefully — `valid_user` looks fine at a glance, but the underscore trips
the `containsInvalidCharacters` check, not the length checks.

The generic `catch { }` at the end (no pattern) binds the error to a local
`error` constant of type `Error` and is required unless every possible error
case is matched explicitly — it's your safety net for errors you didn't
anticipate.

## Three ways to call a throwing function

Beyond `do`/`catch`, Swift has two shorthand forms for different situations:

```swift
// try? -- converts the result to an Optional, discarding the specific error
let maybeValid = try? validateUsername("abc")
print(maybeValid)   // Optional("abc")

let maybeInvalid = try? validateUsername("x")
print(maybeInvalid)   // nil -- we lose *why* it failed

// try! -- force-runs it, crashing the program if it throws
// Only appropriate when failure is truly impossible at that call site
let guaranteed = try! validateUsername("knownGoodValue")
```

`try!` carries the exact same risk as force-unwrapping `!` on an optional —
reach for it only when you can prove, right there in the code, that the call
cannot fail (e.g., validating a hardcoded literal you control).

## Propagating errors upward

A throwing function can call another throwing function with `try` and let
the error propagate to *its* caller instead of handling it locally — this is
how errors travel up a call stack until something is ready to handle them:

```swift
struct User {
    let username: String
}

func createUser(rawName: String) throws -> User {
    let validName = try validateUsername(rawName)   // propagates on failure
    return User(username: validName)
}

do {
    let user = try createUser(rawName: "ab")
    print(user)
} catch {
    print("Could not create user: \(error)")
}
// Could not create user: tooShort(minimum: 3)
```

`createUser` never needed a `do`/`catch` of its own — marking it `throws`
and using `try` was enough to pass the failure straight through.

## The `Result` type

`Result<Success, Failure>` is an enum with two cases, `.success(Success)`
and `.failure(Failure)`. It's most useful when you need to **store or pass
around** an outcome — say, from a completion handler — rather than handle it
immediately with `do`/`catch`:

```swift
func fetchUsername(id: Int) -> Result<String, ValidationError> {
    guard id > 0 else {
        return .failure(.tooShort(minimum: 1))   // reused loosely for the example
    }
    return .success("user\(id)")
}

let result = fetchUsername(id: 5)

switch result {
case .success(let name):
    print("Fetched: \(name)")
case .failure(let error):
    print("Failed: \(error)")
}
// Fetched: user5
```

`Result` also bridges cleanly to throwing code with `get()`, which converts
`.failure` back into a thrown error:

```swift
do {
    let name = try fetchUsername(id: -1).get()
    print(name)
} catch {
    print("get() threw: \(error)")
}
// get() threw: tooShort(minimum: 1)
```

## `throws` vs `Result` vs `Optional` — picking the right tool

| Situation | Use |
|-----------|-----|
| Failure reason doesn't matter, only "did it work" | `Optional` (`nil` on failure) |
| Failure reason matters, and you can handle it synchronously | `throws` + `do`/`try`/`catch` |
| The outcome needs to be stored, returned later, or passed to a completion handler | `Result<Success, Failure>` |
| You're certain failure is impossible at this exact call site | `try!` or force-unwrap (use sparingly) |

## A common trap: forgetting `try` propagates, not converts

```swift
func riskyOperation() throws -> Int {
    throw ValidationError.tooShort(minimum: 1)
}

// This does NOT compile -- calling a throwing function always needs try/try?/try!
// let value = riskyOperation()

// This compiles but crashes immediately if it throws:
// let value = try! riskyOperation()

// The safe, idiomatic version:
if let value = try? riskyOperation() {
    print(value)
} else {
    print("riskyOperation failed")
}
// riskyOperation failed
```

A subtle trap in the other direction: `try?` silently swallows the specific
error. If you find yourself immediately checking *which* error a `try?`
produced, that's a sign you wanted `do`/`catch` instead.

## Exercise

Define an enum `BankError: Error` with cases `insufficientFunds(shortfall:
Double)` and `invalidAmount`. Write a throwing function `withdraw(amount:
Double, from balance: Double) throws -> Double` that returns the new balance,
throwing `.invalidAmount` for a zero or negative amount and
`.insufficientFunds` (with the shortfall) if `amount > balance`. Call it
inside a `do`/`catch` with several test amounts, printing a specific message
for each error case. Then rewrite one call site using `try?` and show how
much information is lost compared to the `catch` version.
