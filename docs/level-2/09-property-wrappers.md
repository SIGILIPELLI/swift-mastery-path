# 09 · Property Wrappers Intro

Property wrappers let you extract a repeated piece of "get/set" behavior —
validation, clamping, storage tricks — into a reusable type, and then apply
it to any property with a single `@` annotation. If you've used SwiftUI's
`@State` or `@Published`, you've already used property wrappers; this
module shows what's actually happening underneath them.

## The repeated-logic problem

Imagine several properties that all need the same clamping behavior:

```swift
struct Settings {
    private var _volume: Int = 50
    var volume: Int {
        get { _volume }
        set { _volume = min(max(newValue, 0), 100) }   // clamp to 0...100
    }

    private var _brightness: Int = 50
    var brightness: Int {
        get { _brightness }
        set { _brightness = min(max(newValue, 0), 100) }   // same clamping logic, duplicated
    }
}
```

The clamping logic is identical for both properties — copy-pasted, which
means a future change (say, a different valid range) has to be made in
every copy. A property wrapper factors this out once.

## Defining a property wrapper

A property wrapper is a struct (or class) annotated `@propertyWrapper`,
with a required `wrappedValue`:

```swift
@propertyWrapper
struct Clamped {
    private var value: Int
    private let range: ClosedRange<Int>

    var wrappedValue: Int {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }

    init(wrappedValue: Int, _ range: ClosedRange<Int>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
}
```

Using it collapses the whole get/set dance to one line per property:

```swift
struct BetterSettings {
    @Clamped(0...100) var volume: Int = 50
    @Clamped(0...100) var brightness: Int = 75
}

var settings = BetterSettings()
settings.volume = 150
print(settings.volume)       // 100 -- clamped automatically

settings.brightness = -20
print(settings.brightness)   // 0 -- clamped automatically
```

`@Clamped(0...100) var volume: Int = 50` desugars to something close to
`private var _volume = Clamped(wrappedValue: 50, 0...100)`, with `volume`
becoming a computed property that reads and writes through `_volume.wrappedValue`
— the `@` syntax is compiler sugar over exactly that pattern.

## A validating wrapper

Property wrappers are equally useful for validation, not just clamping:

```swift
@propertyWrapper
struct NonEmpty {
    private var value: String

    var wrappedValue: String {
        get { value }
        set { value = newValue.isEmpty ? "Unnamed" : newValue }
    }

    init(wrappedValue: String) {
        self.value = wrappedValue.isEmpty ? "Unnamed" : wrappedValue
    }
}

struct Author {
    @NonEmpty var name: String
}

var author = Author(name: "Toni")
print(author.name)   // Toni

author.name = ""
print(author.name)   // Unnamed -- the wrapper silently substitutes a default
```

## `projectedValue`: exposing extra information with `$`

A property wrapper can expose a second value — accessed with a `$` prefix —
alongside the main wrapped value. This is how SwiftUI's `@State` gives you
both the plain value (`count`) and a `Binding` (`$count`):

```swift
import Foundation

@propertyWrapper
struct Trimmed {
    private var value: String = ""
    private(set) var wasTrimmed = false   // tracks whether trimming actually changed anything

    var wrappedValue: String {
        get { value }
        set {
            let trimmedValue = newValue.trimmingCharacters(in: .whitespaces)
            wasTrimmed = trimmedValue != newValue
            value = trimmedValue
        }
    }

    var projectedValue: Bool {
        wasTrimmed
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue
    }
}

struct Comment {
    @Trimmed var text: String = ""
}

var comment = Comment()
comment.text = "  hello world  "
print(comment.text)     // hello world
print(comment.$text)    // true -- accessed via the $ prefix, exposes projectedValue
```

## Built-in property wrappers you'll meet in the wild

You won't write most property wrappers yourself day-to-day — Foundation and
SwiftUI ship several for common patterns:

| Wrapper | Comes from | Purpose |
|---------|------------|---------|
| `@State` | SwiftUI | Local, mutable view state that triggers a UI refresh on change |
| `@Published` | Combine | Marks a property whose changes broadcast to subscribers |
| `@ObservedObject` / `@StateObject` | SwiftUI | Watches an external reference type for changes |
| `@AppStorage` | SwiftUI | Reads/writes a value directly to `UserDefaults` |
| `@Binding` | SwiftUI | A two-way reference to state owned by another view |

Even without a SwiftUI project, understanding the mechanics above means
none of these are "magic" the first time you see them.

## A trap: wrappers and memberwise initializers

Adding a property wrapper to a struct property can change or remove the
automatic memberwise initializer, depending on how the wrapper's own `init`
is defined — this trips people up when refactoring a plain struct into one
using wrappers:

```swift
struct Config {
    @Clamped(0...10) var level: Int = 5
}

// Config(level: 5) still works here because Clamped's init(wrappedValue:_:)
// lets the compiler synthesize a matching memberwise initializer.
let config = Config(level: 5)
print(config.level)   // 5
```

If a wrapper's initializer doesn't line up this cleanly (for example, if it
required extra parameters with no defaults), the memberwise initializer can
disappear entirely, forcing you to write `init` by hand — always check by
trying to construct the type after adding a wrapper.

## Exercise

Write a property wrapper `@propertyWrapper struct Uppercased` that stores a
`String` and always uppercases it on set (so reading `wrappedValue` never
returns anything but uppercase). Apply it to a `var code: String` inside a
`struct Coupon`, set it to `"save20"`, and confirm it prints `"SAVE20"`.
Then add a `projectedValue: Int` to the wrapper that exposes the *original*
(pre-uppercasing) string's character count, and print `coupon.$code` to
confirm it reports the right length.
