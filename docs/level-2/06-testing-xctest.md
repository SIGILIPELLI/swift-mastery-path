# 06 · Testing with XCTest

Every module so far has verified code by eyeballing `print` output. That
doesn't scale — once a project has more than a handful of functions, you
need something that checks your assumptions automatically, every time you
change the code. **XCTest** is Apple's built-in testing framework, and it
works the same way whether you're testing an iOS app or a Swift Package.

## Anatomy of a test

A test file imports `XCTest`, defines a class subclassing `XCTestCase`, and
puts each test in a method whose name starts with `test`:

```swift
import XCTest
@testable import MyLibrary   // gives the test target access to internal (non-public) code

final class MathUtilsTests: XCTestCase {

    func testAddition() {
        let result = MathUtils.add(2, 3)
        XCTAssertEqual(result, 5)
    }

    func testAdditionWithNegatives() {
        let result = MathUtils.add(-2, 3)
        XCTAssertEqual(result, 1)
    }
}
```

The `test` prefix isn't a style choice — it's how the XCTest runner
discovers which methods are actual tests versus ordinary helper methods on
the class.

## The assertion family

`XCTAssertEqual` is the one you'll use most, but XCTest ships a full set of
assertions for different situations:

```swift
func testAssertionVariety() {
    XCTAssertEqual(2 + 2, 4)
    XCTAssertNotEqual(2 + 2, 5)
    XCTAssertTrue(3 > 2)
    XCTAssertFalse(2 > 3)
    XCTAssertNil(Int("not a number"))
    XCTAssertNotNil(Int("42"))

    let numbers = [1, 2, 3]
    XCTAssertTrue(numbers.contains(2))

    // Custom failure message -- shown in the test log if the assertion fails
    XCTAssertEqual(numbers.count, 3, "Expected exactly three numbers")
}
```

Each assertion, when it fails, reports the exact file and line — that's
what lets you jump straight to the broken expectation instead of hunting
through `print` output.

## Testing thrown errors

Because Level 2's error-handling module introduced `throws`, XCTest has
matching tools for it:

```swift
enum ValidationError: Error, Equatable {
    case tooShort
}

func validateUsername(_ name: String) throws -> String {
    guard name.count >= 3 else { throw ValidationError.tooShort }
    return name
}

final class ValidationTests: XCTestCase {

    func testValidUsernamePasses() throws {
        // "try" is required here too -- this test method itself can throw
        let result = try validateUsername("alice")
        XCTAssertEqual(result, "alice")
    }

    func testShortUsernameThrows() {
        XCTAssertThrowsError(try validateUsername("ab")) { error in
            XCTAssertEqual(error as? ValidationError, .tooShort)
        }
    }

    func testValidUsernameDoesNotThrow() {
        XCTAssertNoThrow(try validateUsername("alice"))
    }
}
```

`XCTAssertThrowsError` fails the test if the expression *doesn't* throw —
and its trailing closure receives the actual error so you can assert it's
the *specific* failure you expected, not just "something went wrong."

## `setUp` and `tearDown`

When multiple tests need the same fresh starting state, `setUp()` runs
before every test method, and `tearDown()` after — this avoids duplicating
setup code and, more importantly, guarantees each test starts from a clean
slate independent of the others:

```swift
final class ShoppingCartTests: XCTestCase {
    var cart: ShoppingCart!

    override func setUp() {
        super.setUp()
        cart = ShoppingCart()   // brand-new cart before every single test
    }

    override func tearDown() {
        cart = nil
        super.tearDown()
    }

    func testEmptyCartHasZeroTotal() {
        XCTAssertEqual(cart.total, 0)
    }

    func testAddingItemUpdatesTotal() {
        cart.add(price: 9.99)
        XCTAssertEqual(cart.total, 9.99, accuracy: 0.001)
    }
}
```

`accuracy:` matters for `Double` comparisons — floating-point arithmetic can
introduce tiny rounding errors, so `XCTAssertEqual(a, b)` on two `Double`s
that are "conceptually equal" can fail by a fraction of a cent. `accuracy:`
declares an acceptable margin instead of demanding bit-for-bit equality.

## Why tests must be independent

A common beginner trap is writing tests that depend on execution order —
for example, a `testAdd` that leaves items in a shared cart, and a
`testCount` that assumes those items are still there. XCTest does **not**
guarantee methods run in the order they're written, and running tests in
isolation (e.g., re-running just one failing test) is a normal workflow —
if `testCount` secretly depends on `testAdd` running first, it will fail
unpredictably. `setUp`/`tearDown` resetting shared state is exactly what
prevents this.

## Running tests

Inside an Xcode project, `Cmd+U` runs the whole suite. For a Swift Package
(covered in [Module 8](08-swift-package-manager.md)), it's a single command:

```bash
swift test
```

```text
Test Suite 'All tests' started at ...
Test Suite 'MathUtilsTests' started at ...
Test Case 'MathUtilsTests.testAddition' passed (0.001 seconds).
Test Case 'MathUtilsTests.testAdditionWithNegatives' passed (0.001 seconds).
Test Suite 'MathUtilsTests' passed.
Executed 2 tests, with 0 failures in 0.002 seconds
```

## Cheat sheet

| Assertion | Checks |
|-----------|--------|
| `XCTAssertEqual(a, b)` | `a == b` |
| `XCTAssertEqual(a, b, accuracy: e)` | `abs(a - b) <= e` (for floating-point) |
| `XCTAssertTrue(x)` / `XCTAssertFalse(x)` | Boolean condition |
| `XCTAssertNil(x)` / `XCTAssertNotNil(x)` | Optional is/isn't `nil` |
| `XCTAssertThrowsError(try f())` | Expression throws |
| `XCTAssertNoThrow(try f())` | Expression does not throw |
| `setUp()` / `tearDown()` | Runs before/after every test method |

## Exercise

Write a small `struct Calculator` with `add`, `subtract`, and a throwing
`divide(_:by:)` that throws a `DivisionError.byZero` error when the divisor
is `0`. Then write `final class CalculatorTests: XCTestCase` with: a test
for `add`, a test for `subtract`, a test that `divide` returns the correct
result for valid input using `XCTAssertEqual`, and a test that
`divide(_:by: 0)` throws using `XCTAssertThrowsError`. Run `swift test` (once
you've set up a package as shown in [Module 8](08-swift-package-manager.md))
and confirm all four pass.
