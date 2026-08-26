# 08 · Interop with C/Objective-C

Swift was designed to interoperate with C and Objective-C from day one —
most of Apple's platform frameworks are still Objective-C or C underneath.
This module covers calling C functions directly, working with raw
pointers, and the Objective-C runtime features (`NSObject`, `@objc`, KVC)
that Swift code can still opt into.

## Calling C functions directly

C standard library and system functions are available in Swift without any
wrapping — `Foundation` (and the platform SDK generally) exposes them as
plain Swift function calls:

```swift
import Foundation

let random = arc4random_uniform(100)
print("Random C function call returned a value in [0,100):", random < 100)
```

Output:

```
Random C function call returned a value in [0,100): true
```

`arc4random_uniform` is a C function from `<stdlib.h>` — Swift calls it
exactly as written in C, no bridging layer needed, because the Swift
toolchain auto-generates a Swift-callable interface from the system's C
headers (a Clang importer, working the opposite direction from a manual
bridging header).

## C strings

C represents strings as `char *` (null-terminated byte buffers); Swift
bridges these through `UnsafeMutablePointer<CChar>`/`UnsafePointer<CChar>`
and `String(cString:)`:

```swift
let cString = strdup("hello from C")
if let cString {
    let swiftString = String(cString: cString)
    print("Bridged C string:", swiftString)
    free(cString)   // strdup allocates with malloc; Swift won't free it for you
}
```

Output:

```
Bridged C string: hello from C
```

`strdup` allocates memory with C's `malloc` — Swift's ARC has no idea this
pointer exists, so it will never be freed automatically; the explicit
`free(cString)` is not optional cleanup, it's required to avoid a leak.

## Working with raw buffers

`withUnsafeBufferPointer` gives temporary, safe-scoped access to a Swift
array's contiguous storage as a C-style pointer + count, useful for calling
C APIs that expect exactly that shape:

```swift
func sumBuffer(_ pointer: UnsafePointer<Int32>, count: Int) -> Int32 {
    var total: Int32 = 0
    for i in 0..<count {
        total += pointer[i]
    }
    return total
}

let values: [Int32] = [1, 2, 3, 4, 5]
let total = values.withUnsafeBufferPointer { buffer -> Int32 in
    sumBuffer(buffer.baseAddress!, count: buffer.count)
}
print("Sum via C-style pointer access:", total)
```

Output:

```
Sum via C-style pointer access: 15
```

The pointer handed to the closure is only valid for the duration of that
closure call — storing it anywhere and using it after `withUnsafeBufferPointer`
returns is undefined behavior, since the array is free to move or
deallocate that storage afterward.

## Objective-C interop: `NSObject`, `@objc`, dynamic dispatch

A Swift class that needs to participate in Objective-C runtime features
(KVC/KVO, target-action, `NSNotificationCenter` selectors) subclasses
`NSObject` and marks members `@objc`:

```swift
import ObjectiveC

class Greeter: NSObject {
    @objc dynamic var name: String = "World"

    @objc func greet() -> String {
        "Hello, \(name)!"
    }
}

let greeter = Greeter()
greeter.name = "Swift"
print(greeter.greet())
```

Output:

```
Hello, Swift!
```

`dynamic` on `name` forces Objective-C-style dynamic dispatch (a message
send) for that property, which is what allows runtime features like KVO
to intercept reads and writes — a plain Swift `var` (even `@objc`, without
`dynamic`) uses Swift's normal static/vtable dispatch, which KVO can't hook
into.

## `NSString` bridging

`String` and `NSString` bridge transparently in both directions — the same
underlying storage, exposed through whichever API you're calling:

```swift
let nsString: NSString = "Objective-C string"
let swiftFromNS: String = nsString as String
print("Bridged NSString:", swiftFromNS)
print("Uppercase via NSString API:", nsString.uppercased)
```

Output:

```
Bridged NSString: Objective-C string
Uppercase via NSString API: OBJECTIVE-C STRING
```

## Key-Value Coding (KVC)

`value(forKey:)` is a purely Objective-C-runtime feature — it looks up a
property by its string name at runtime rather than through Swift's
compile-time member access, which is why it requires `@objc`/`NSObject`:

```swift
let value = greeter.value(forKey: "name") as? String
print("KVC value(forKey:):", value ?? "nil")
```

Output:

```
KVC value(forKey:): Swift
```

## Swift-specific traps

- **`@objc` alone doesn't enable KVO — you need `dynamic` too.** A `@objc
  var` without `dynamic` is still statically dispatched by the Swift
  compiler for calls it can see at compile time, which bypasses the
  message-send mechanism KVO relies on to intercept changes.
- **Memory from C allocators (`malloc`, `strdup`) is never managed by
  ARC** — every `malloc`/`strdup`/similar needs an explicit matching
  `free`; forgetting one is a genuine leak, not a Swift-caught bug.
- **`Unsafe*Pointer` types provide zero bounds checking** — indexing past
  the buffer's actual length (as in `sumBuffer` above, if `count` were
  wrong) is undefined behavior, potentially reading or corrupting unrelated
  memory, not a clean crash.
- **`value(forKey:)` returns `Any?`**, so the `as? String` cast can fail
  silently (yielding `nil`) if the property name is misspelled or the
  actual type doesn't match — unlike a compile-time-checked Swift property
  access, a typo here is only caught at runtime, if at all.

## Cheat sheet

| Need | Approach |
|------|----------|
| Call a C function | Just call it — the Clang importer exposes it directly |
| Bridge a C string | `String(cString:)`, and `free()` if it was heap-allocated |
| Pass a Swift array to a C-style API | `array.withUnsafeBufferPointer { ... }` |
| Enable KVO/KVC on a property | `@objc dynamic var` on an `NSObject` subclass |
| Bridge `String` ↔ `NSString` | `as String` / `as NSString`, transparently |

## Exercise

Write a small C-interop demo that uses `UnsafeMutablePointer<Int32>` (via
`UnsafeMutablePointer<Int32>.allocate(capacity:)`) to manually allocate a
buffer of 10 integers, fill it with squares (`0, 1, 4, 9, ...`), read them
back into a Swift `[Int32]`, and then explicitly `deallocate()` the buffer.
Then write an `NSObject` subclass `Counter` with an `@objc dynamic var
count: Int = 0` and a method that increments it; use `addObserver`/KVO (or
`value(forKey:)` before and after incrementing) to demonstrate the runtime
can observe the property change without any Swift-level callback wired up
by you directly.
