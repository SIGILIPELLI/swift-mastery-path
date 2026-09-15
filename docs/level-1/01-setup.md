---
description: "Setup & First Program — Swift ships built-in on macOS via Xcode (or the smaller Xcode Command Line Tools), and is available as a standalone toolchain on…"
---

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

## How It Actually Works

When you run `swift file.swift` or hit Run in Xcode, several distinct tools chain
together before a single instruction executes:

- **The Swift compiler (`swiftc`)** parses your source into an AST, then lowers it
  to **SIL** (Swift Intermediate Language) — a Swift-specific IR that's where the
  compiler enforces exclusivity checking and performs whole-module optimizations
  (generic specialization, inlining, ARC optimization passes) before handing off to
  LLVM IR and finally to machine code via LLVM's backend.
- **The Swift runtime** is a separate shared library (`libswiftCore`) linked into
  every binary. It provides metadata for types (used for reflection, `String`
  description, and dynamic casts like `as?`), the memory allocator hooks for
  reference counting, and the existential container machinery for protocol types.
- **Package/toolchain resolution**: `swift build`, Xcode, and SwiftPM all resolve
  which toolchain (Swift version, SDK, target triple) to use before invoking the
  same underlying `swiftc` — this is why `swift --version` and your Xcode's
  "Swift Language Version" setting must agree, or you'll see ABI-mismatch style
  errors.
- On Apple platforms, the compiled binary also embeds a **Swift ABI stability**
  marker; since Swift 5, the compiler emits code that talks to a fixed runtime ABI
  baked into the OS, which is why apps don't have to bundle the whole Swift runtime
  themselves on modern OS versions (it ships in the OS instead).

## 🔀 See this in another language

- [Kotlin — Setup & First Program](https://sigilipelli.github.io/kotlin-mastery-path/level-1/01-setup/)
- [Shell/Bash — Setup & First Script](https://sigilipelli.github.io/shell-mastery-path/level-1/01-setup/)
- [C — Setup & First Program](https://sigilipelli.github.io/c-mastery-path/level-1/01-setup/)

## Exercise

Write a file `greeter.swift` that prints a greeting for three different
names, one per line, using `print`. Run it two ways: directly with
`swift greeter.swift`, and compiled with `swiftc greeter.swift -o greeter`
followed by `./greeter`.
