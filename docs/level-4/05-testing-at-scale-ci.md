# 05 · Testing at Scale & CI

A test suite that takes 45 minutes and can't tell you *why* it failed stops
being useful long before a codebase gets large. This module covers
structuring tests so they scale — shared fixtures, parallelization,
flakiness triage — and wiring them into GitHub Actions CI.

!!! note "Environment note for this module"
    `XCTest` needs full Xcode to run, unavailable on this CLT-only
    machine (as in Level 3's testing module). The test code below is
    hand-reviewed for correct `XCTest` API usage; the CI YAML is valid
    GitHub Actions syntax that would run on a macOS runner with Xcode
    installed, which this sandbox is not.

## Shared setup with `XCTestCase.setUp`/`tearDown`

Repeating the same fixture construction in every test method doesn't scale
past a handful of tests. `setUp`/`tearDown` run before/after *each* test
method in the class, keeping fixtures fresh and isolated:

```swift
import XCTest

final class InventoryTests: XCTestCase {
    var inventory: Inventory!

    override func setUp() {
        super.setUp()
        inventory = Inventory()   // fresh instance per test, no leftover state
    }

    override func tearDown() {
        inventory = nil
        super.tearDown()
    }

    func testAddingIncreasesCount() {
        inventory.add(sku: "WIDGET-1", quantity: 5)
        XCTAssertEqual(inventory.count(for: "WIDGET-1"), 5)
    }

    func testRemovingDecreasesCount() {
        inventory.add(sku: "WIDGET-1", quantity: 5)
        inventory.remove(sku: "WIDGET-1", quantity: 2)
        XCTAssertEqual(inventory.count(for: "WIDGET-1"), 3)
    }
}
```

Both tests get a brand-new `Inventory()` — if one test forgot to reset
state, it couldn't leak into and silently affect the next one, which is
what makes tests safe to run in any order or in parallel.

## Parallel test execution

`xcodebuild test -parallel-testing-enabled YES` (or the equivalent Swift
Package Manager flag) runs test classes across multiple simulators/processes
concurrently. This is where `setUp`/`tearDown`-based isolation pays off —
tests sharing mutable global state (a singleton, a shared file path) will
intermittently fail or corrupt each other's results once run in parallel,
even if they passed reliably in sequence.

```bash
swift test --parallel
```

## Structuring larger suites: test plans and tags

For codebases with hundreds of tests, an Xcode **test plan** (a `.xctestplan`
JSON file) lets you group tests into named configurations — a fast
"smoke" plan for pre-commit hooks, a full plan for CI, a plan targeting
just one feature area for local iteration:

```json
{
  "configurations": [
    {
      "name": "Smoke",
      "options": {}
    }
  ],
  "testTargets": [
    {
      "target": { "name": "InventoryTests" },
      "selectedTests": ["InventoryTests/testAddingIncreasesCount"]
    }
  ]
}
```

## Diagnosing flaky tests

A test that fails intermittently is worse than one that always fails — it
erodes trust in the whole suite ("just rerun it"). Common causes specific
to Swift/Apple platforms:

- **Unawaited async work** — a test that doesn't `await` everything it
  starts can finish before a background `Task` completes, passing or
  failing based on scheduling luck.
- **Shared mutable singletons** (Module 06, Level 3's testability section)
  — two tests touching the same global state race when run in parallel.
- **Time-based assertions** (`Date()` comparisons, `sleep`-based waits)
  — assertions like "this happened within 100ms" are inherently flaky on
  loaded CI hardware; assert on logical completion (an `XCTestExpectation`
  fulfilling) instead of wall-clock timing wherever possible.

## Wiring tests into GitHub Actions

A minimal CI workflow that builds and tests a Swift package on every push:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_15.4.app
      - name: Build
        run: swift build
      - name: Run tests
        run: swift test --parallel
```

`runs-on: macos-14` is required for any job needing the iOS SDK/simulator
or XCTest — `ubuntu-latest` runners can build and test pure-Swift packages
(no UIKit/SwiftUI/XCTest-on-Apple-only-APIs) but not iOS-targeted code.

## Failing the build on coverage regressions

Adding a coverage gate catches PRs that add code without adding tests:

```yaml
      - name: Run tests with coverage
        run: swift test --enable-code-coverage
      - name: Check coverage threshold
        run: |
          xcrun llvm-cov report \
            .build/debug/PackageTests.xctest/Contents/MacOS/PackageTests \
            -instr-profile .build/debug/codecov/default.profdata \
            -summary-only
```

A hard coverage percentage gate is a blunt instrument — pairing it with
required review on the diff (so a human judges *which* lines went
untested, not just the aggregate number) tends to catch more real gaps
than a threshold alone.

## Swift-specific traps

- **`setUp`/`tearDown` must call `super`** — omitting `super.setUp()` or
  `super.tearDown()` can skip framework-level setup/teardown work XCTest
  itself relies on.
- **`swift test --parallel` parallelizes across test *classes*, not
  individual test methods within one class** by default — a class with one
  extremely slow test method won't get faster just by adding the flag.
- **CI runners are slower and more loaded than a local machine** — timing
  assumptions that hold locally (a `Task.sleep` "should be enough" for
  something to finish) are a common source of tests that pass locally and
  flake in CI specifically.
- **`macos-14` runners come with a specific pinned Xcode version** that
  changes over time — pinning `xcode-select` explicitly (as in the
  workflow above) prevents a GitHub Actions image update from silently
  changing which Swift/SDK version your CI builds against.

## Cheat sheet

| Need | Tool |
|------|------|
| Fresh fixture per test | `override func setUp()` / `tearDown()` |
| Run tests concurrently | `swift test --parallel` / `-parallel-testing-enabled YES` |
| Group tests into named suites | `.xctestplan` test plans |
| CI on every push/PR | GitHub Actions workflow, `runs-on: macos-14` |
| Catch untested new code | `--enable-code-coverage` + `llvm-cov` |

## Exercise

Write an `XCTestCase` subclass `RateLimiterTests` with `setUp`/`tearDown`
constructing a fresh `RateLimiter(maxRequests: 3, perSeconds: 1)` for every
test. Write three test methods: one confirming the first 3 calls to
`allow()` succeed, one confirming a 4th call within the same window fails,
and one confirming a call succeeds again after advancing your test's
injected clock (reuse the `ClockProviding`/`FixedClock` pattern from Level
3's testing module) past the window. Then sketch the GitHub Actions job
that would run this suite on every pull request targeting `main`.
