# 08 · Swift Package Manager

Every project so far has been a single `.swift` file compiled directly with
`swiftc`. That works for small scripts, but real projects need multiple
source files, external dependencies, and a way to run automated tests —
**Swift Package Manager (SPM)** is the built-in tool for all three, and it's
what makes the [Weather CLI project](10-project-weather-cli.md) possible.

## Creating a package

```bash
mkdir GreetingKit && cd GreetingKit
swift package init --type executable
```

This scaffolds a standard layout:

```text
GreetingKit/
    Package.swift
    Sources/
        GreetingKit/
            GreetingKit.swift
    Tests/
        GreetingKitTests/
            GreetingKitTests.swift
```

## `Package.swift`: the manifest

`Package.swift` is itself Swift code — it describes the package's name,
supported platforms, dependencies, and build targets:

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "GreetingKit",
    products: [
        // What this package exposes to other packages/executables
        .executable(name: "GreetingKit", targets: ["GreetingKit"])
    ],
    dependencies: [
        // External packages would be listed here, e.g.:
        // .package(url: "https://github.com/apple/swift-argument-parser", from: "1.3.0")
    ],
    targets: [
        .executableTarget(
            name: "GreetingKit",
            dependencies: []
        ),
        .testTarget(
            name: "GreetingKitTests",
            dependencies: ["GreetingKit"]
        )
    ]
)
```

A **target** is a group of source files compiled together. `.executableTarget`
produces a runnable binary; `.target` produces a library other targets can
import; `.testTarget` holds XCTest code and can only be run, never shipped.

## Multi-file organization

Everything inside `Sources/GreetingKit/` is compiled into one module — you
can freely split code across files without any `import` between them, as
long as they're in the same target:

```swift
// Sources/GreetingKit/Greeter.swift
struct Greeter {
    let name: String

    func greet() -> String {
        "Hello, \(name)!"
    }
}
```

```swift
// Sources/GreetingKit/main.swift
let greeter = Greeter(name: "World")   // no import needed -- same target as Greeter.swift
print(greeter.greet())
```

This is the practical reason SPM projects exist: once a project outgrows a
single file (models, networking, business logic, CLI parsing all mixed
together), splitting by responsibility into separate files under `Sources/`
keeps it navigable, while the compiler still treats it as one cohesive
module.

## Building and running

```bash
swift build              # compiles the package
swift run                # builds (if needed) and runs the executable
swift run GreetingKit    # explicit target name, useful with multiple executables
```

```text
$ swift run
Building for debugging...
Build complete!
Hello, World!
```

## Adding tests

`Tests/GreetingKitTests/GreetingKitTests.swift` uses exactly the XCTest
patterns from [Module 6](06-testing-xctest.md):

```swift
import XCTest
@testable import GreetingKit

final class GreetingKitTests: XCTestCase {
    func testGreetUsesName() {
        let greeter = Greeter(name: "Swift")
        XCTAssertEqual(greeter.greet(), "Hello, Swift!")
    }
}
```

```bash
swift test
```

```text
Test Suite 'All tests' started at ...
Test Case 'GreetingKitTests.testGreetUsesName' passed (0.001 seconds).
Executed 1 test, with 0 failures in 0.001 seconds
```

`@testable import` is what makes `internal` (the default access level)
declarations visible to the test target — without it, only `public` types
and members would be reachable from `GreetingKitTests`.

## Adding a dependency

Real projects often depend on other packages. Adding one means listing it
under `dependencies:` in `Package.swift` and referencing it from whichever
target actually uses it:

```swift
let package = Package(
    name: "GreetingKit",
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser", from: "1.3.0")
    ],
    targets: [
        .executableTarget(
            name: "GreetingKit",
            dependencies: [
                .product(name: "ArgumentParser", package: "swift-argument-parser")
            ]
        )
    ]
)
```

`swift build` resolves and downloads the dependency automatically the next
time it runs, recording the exact resolved versions in `Package.resolved` —
commit that file so everyone building the project gets identical dependency
versions.

## Access control across a package

By default, types and members are `internal` — visible anywhere inside the
same module, but invisible to other modules (including packages that
depend on yours). Mark anything meant to be used by external code `public`:

```swift
// Only usable inside the GreetingKit module itself:
struct InternalHelper { }

// Usable by any package that imports GreetingKit:
public struct Greeter {
    public let name: String
    public init(name: String) { self.name = name }   // public init is required too!
    public func greet() -> String { "Hello, \(name)!" }
}
```

A common trap: marking a struct `public` but forgetting its `init` — without
an explicit `public init`, external code can't construct the type at all,
even though the type itself is visible.

## Cheat sheet

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

## Exercise

Run `swift package init --type executable` for a package named
`TemperatureKit`. Split the logic across two files in `Sources/
TemperatureKit/`: a `Converter.swift` with a `struct TemperatureConverter`
exposing `celsiusToFahrenheit(_:)` and `fahrenheitToCelsius(_:)`, and keep
`main.swift` as a tiny CLI that converts a hardcoded value and prints it.
Then add a `Tests/TemperatureKitTests/` test file with at least two
`XCTAssertEqual` cases covering both conversion directions, and confirm
`swift test` passes.
