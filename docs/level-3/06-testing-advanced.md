# 06 · Testing Advanced

Level 2 covered basic `XCTest` assertions. This module covers the patterns
that make tests trustworthy in larger codebases: dependency injection for
testability, mocking protocols, async test support, and measuring
performance.

!!! note "Environment note for this module"
    `XCTest` requires a full Xcode installation to run (`xcodebuild` or the
    `XCTest` runtime aren't part of the Command Line Tools alone), and this
    machine has only the CLT. The test code below is hand-reviewed for
    correct, current `XCTest`/Swift Testing API usage — run it with `swift
    test` on a machine with full Xcode to confirm.

## Designing for testability: dependency injection

Code that reaches directly for a global (`URLSession.shared`, a singleton
database) is hard to test because you can't substitute a fake. Depending on
a protocol instead lets tests inject a double:

```swift
protocol WeatherFetching {
    func fetchTemperature(for city: String) async throws -> Double
}

struct LiveWeatherService: WeatherFetching {
    func fetchTemperature(for city: String) async throws -> Double {
        // in reality, a real network call
        return 21.5
    }
}

struct WeatherPresenter {
    let service: WeatherFetching

    func summary(for city: String) async throws -> String {
        let temp = try await service.fetchTemperature(for: city)
        return "\(city): \(temp)°C"
    }
}
```

`WeatherPresenter` depends on the `WeatherFetching` protocol, not on
`LiveWeatherService` directly — production code injects the live service,
tests inject a fake.

## A mock/stub conforming to the same protocol

```swift
import XCTest

final class MockWeatherService: WeatherFetching {
    var stubbedTemperature: Double = 0
    var fetchCallCount = 0

    func fetchTemperature(for city: String) async throws -> Double {
        fetchCallCount += 1
        return stubbedTemperature
    }
}

final class WeatherPresenterTests: XCTestCase {
    func testSummaryFormatsTemperature() async throws {
        let mock = MockWeatherService()
        mock.stubbedTemperature = 18.0
        let presenter = WeatherPresenter(service: mock)

        let result = try await presenter.summary(for: "Berlin")

        XCTAssertEqual(result, "Berlin: 18.0°C")
        XCTAssertEqual(mock.fetchCallCount, 1)
    }
}
```

The mock tracks `fetchCallCount` in addition to returning a stubbed value —
asserting on both the *result* and the *interaction* (was the dependency
called, and how many times) catches bugs that checking the return value
alone would miss, like an accidental double-fetch.

## Testing errors

A dependency's failure path deserves its own test — throw from the mock and
assert the error propagates or is handled correctly:

```swift
struct WeatherError: Error, Equatable { let message: String }

final class FailingWeatherService: WeatherFetching {
    func fetchTemperature(for city: String) async throws -> Double {
        throw WeatherError(message: "network unavailable")
    }
}

func testSummaryPropagatesError() async {
    let presenter = WeatherPresenter(service: FailingWeatherService())
    do {
        _ = try await presenter.summary(for: "Berlin")
        XCTFail("Expected an error to be thrown")
    } catch let error as WeatherError {
        XCTAssertEqual(error.message, "network unavailable")
    } catch {
        XCTFail("Unexpected error type: \(error)")
    }
}
```

`XCTFail("Expected an error to be thrown")` on the success path is
important — without it, a test that's supposed to verify failure would
silently pass if the throw stopped happening (say, after a refactor).

## `XCTestExpectation` for callback-based async code

Not everything is `async`/`await` yet — legacy completion-handler APIs need
`XCTestExpectation` to wait for a callback:

```swift
func testLegacyCallbackAPI() {
    let expectation = expectation(description: "callback fires")

    legacyFetchTemperature(for: "Paris") { temperature in
        XCTAssertEqual(temperature, 15.0)
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 2.0)
}
```

`wait(for:timeout:)` blocks the test until `fulfill()` is called or the
timeout elapses — a test that never calls `fulfill()` fails loudly with a
timeout rather than hanging forever.

## Performance tests

`measure` runs a block multiple times and reports timing, useful for
catching accidental performance regressions:

```swift
func testSortPerformance() {
    let numbers = (0..<10_000).shuffled()
    measure {
        _ = numbers.sorted()
    }
}
```

## Swift-specific traps

- **`async` test methods need `async` in their signature** (`func
  testFoo() async throws`) — XCTest recognizes this and awaits the test
  body automatically; forgetting `async` on a test that awaits inside it is
  a compile error, which is a helpful guardrail.
- **A mock class needs to actually conform to the protocol**, not just
  duck-type it — Swift's protocol conformance is nominal, so a class that
  merely happens to have a matching method won't satisfy
  `WeatherFetching` without an explicit `: WeatherFetching`.
- **`XCTestExpectation` and `async`/`await` don't need to be mixed** — a
  fully `async` API doesn't need expectations at all; reach for them only
  when bridging to genuinely callback-based code.
- **`measure` results are noisy on shared/virtualized hardware** (like most
  CI runners) — treat performance test failures as a signal to
  investigate, not an absolute pass/fail gate, unless you've tuned the
  baseline carefully for that environment.

## Cheat sheet

| Need | Tool |
|------|------|
| Substitute a fake dependency | A protocol + a test-only conforming type |
| Assert an interaction happened | A counter/flag property on the mock |
| Test an `async` function | `func testX() async throws { ... }` |
| Wait for a completion handler | `XCTestExpectation` + `wait(for:timeout:)` |
| Catch performance regressions | `measure { ... }` |

## Exercise

Write a protocol `ClockProviding` with `func now() -> Date`, a
`LiveClock` conforming implementation using `Date()`, and a `FixedClock`
test double that always returns a date you pass into its initializer. Write
a `SessionTimeout` type that depends on `ClockProviding` and exposes `func
isExpired(since start: Date, after seconds: TimeInterval) -> Bool`. Write
two `XCTestCase` tests — one proving `isExpired` returns `true` once the
fixed clock has advanced past the timeout, one proving it returns `false`
just before.
