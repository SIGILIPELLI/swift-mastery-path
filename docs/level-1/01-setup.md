# 01 · Setup & First Program

## Install the Swift toolchain

Swift ships built-in on macOS via **Xcode** (or the smaller Xcode Command Line
Tools), and is available as a standalone toolchain on Linux and Windows.

```bash
# macOS -- install the command line tools (includes swiftc, swift)
xcode-select --install

# Or install full Xcode from the App Store for iOS/macOS app development

# Linux (Ubuntu) -- download a toolchain from swift.org, or use swiftly:
curl -s https://swift.org/install.sh | bash

# Windows: installer from https://www.swift.org/install/windows/
```

Verify the install:

```bash
swift --version
# Swift version 5.10 (or later)
```

`swiftc` is the compiler; `swift` runs the interactive REPL or executes
source files directly without a separate compile step.

## The `swift` REPL

The REPL (Read-Eval-Print Loop) is the fastest way to try out small snippets:

```bash
swift
```

```swift
1> print("Hello, world!")
Hello, world!
2> let x = 10
x: Int = 10
3> x * 2
$R0: Int = 20
4> :quit
```

Every line you type is evaluated immediately and its result printed — great
for experimenting with a new API before committing it to a file.

## Running a `.swift` file

Create `hello.swift`:

```swift
// hello.swift
print("Hello, world!")
```

Run it directly — Swift compiles in memory and executes immediately:

```bash
swift hello.swift
# Hello, world!
```

Notice there's no `public class` wrapper and no explicit `main` function —
top-level code in a single file *is* the entry point. This is one of Swift's
biggest ergonomic wins for scripting and learning.

## Compiling to a binary

For anything you'll run more than once, compile an optimized executable:

```bash
swiftc hello.swift -o hello
# produces the "hello" binary

./hello
# Hello, world!
```

`swiftc -O hello.swift -o hello` adds optimizations, closer to what a release
build does.

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `print(...)` | Writes text followed by a newline to standard output. |
| No semicolons required | A newline ends a statement (semicolons are optional, used only to put two statements on one line). |
| No `main` function needed | Top-level statements in a single-file script run top to bottom, in order. |
| `//` | Starts a single-line comment; `/* ... */` for multi-line. |

## Choosing an editor

**Xcode** (macOS only, free) gives the richest experience — autocomplete,
inline diagnostics, a debugger, and Swift Playgrounds for live-executing
snippets as you type. On Linux or for lightweight editing anywhere, **VS
Code** with the official "Swift" extension (powered by SourceKit-LSP) works
well. Either is fine for this course — pick one and move on.

## Exercise

Write a file `greeter.swift` that prints a greeting for three different
names, one per line, using `print`. Run it two ways: directly with
`swift greeter.swift`, and compiled with `swiftc greeter.swift -o greeter`
followed by `./greeter`.
