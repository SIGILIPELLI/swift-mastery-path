# 03 · SwiftUI Fundamentals

Level 3's iOS Fundamentals module covered the app lifecycle and touched
SwiftUI's `NavigationStack`. This module goes deeper into SwiftUI itself:
state management (`@State`, `@Binding`, `@Observable`), view composition,
and the declarative rendering model that ties them together.

!!! note "Environment note for this module"
    SwiftUI views need a simulator or device to actually render, and this
    machine has only the Xcode Command Line Tools (no iOS SDK, no
    simulator). The code below is hand-reviewed for correct, current
    SwiftUI API usage (targeting the `@Observable` macro introduced in
    iOS 17 / Swift 5.9) — verify it by running it in Xcode's canvas or the
    simulator.

## The declarative model: a view is a function of state

A SwiftUI `View` describes *what* the UI should look like for the current
state — not a sequence of imperative UI-mutating calls. When state changes,
SwiftUI recomputes the affected part of the view tree:

```swift
import SwiftUI

struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 16) {
            Text("Count: \(count)")
                .font(.title)
            Button("Increment") {
                count += 1
            }
        }
    }
}
```

`@State` marks `count` as SwiftUI-managed storage: mutating it from inside
the `Button`'s action closure automatically triggers `body` to be
recomputed, redrawing `Text` with the new value. `@State` is meant for
value types owned by exactly this view — never share a `@State` property's
underlying storage with another view directly.

## `@Binding`: sharing mutable state with a child view

A child view that needs to both read *and write* a parent's state receives
a `Binding` — a two-way reference — rather than a plain value:

```swift
struct ToggleRow: View {
    let title: String
    @Binding var isOn: Bool

    var body: some View {
        Toggle(title, isOn: $isOn)
    }
}

struct SettingsView: View {
    @State private var notificationsEnabled = true

    var body: some View {
        ToggleRow(title: "Notifications", isOn: $notificationsEnabled)
    }
}
```

The `$` prefix on `$notificationsEnabled` projects the `@State` property
into a `Binding<Bool>` — `ToggleRow` can flip it, and `SettingsView`'s own
`@State` updates in lockstep, because they refer to the same underlying
storage rather than a copy.

## `@Observable`: shared reference-type state

For state that outlives a single view (shared across a whole screen, or
persisted across navigation), an `@Observable` class replaces the older
`ObservableObject`/`@Published` combination with fewer annotations:

```swift
import Observation

@Observable
final class CartModel {
    var items: [String] = []
    var total: Double = 0

    func add(_ item: String, price: Double) {
        items.append(item)
        total += price
    }
}

struct CartView: View {
    let cart: CartModel

    var body: some View {
        VStack {
            List(cart.items, id: \.self) { Text($0) }
            Text("Total: \(cart.total, format: .currency(code: "USD"))")
            Button("Add Coffee") {
                cart.add("Coffee", price: 4.50)
            }
        }
    }
}
```

`@Observable` tracks which properties a given view's `body` actually reads
and only invalidates that view when one of *those* properties changes —
more granular than `ObservableObject`, which invalidated on any
`@Published` change regardless of whether the view read it.

## View composition and modifiers

SwiftUI views compose by nesting, and modifiers (`.padding()`, `.font()`,
`.background()`) each return a new, wrapped view rather than mutating one
in place:

```swift
struct Badge: View {
    let text: String

    var body: some View {
        Text(text)
            .font(.caption.bold())
            .padding(.horizontal, 8)
            .padding(.vertical, 4)
            .background(Color.blue.opacity(0.2))
            .clipShape(Capsule())
    }
}
```

Modifier order matters — `.padding()` before `.background()` pads inside
the colored capsule; swapping the order would pad *outside* it, changing
the visual result entirely, since each modifier wraps the view produced by
the previous one.

## Lists and `ForEach`

Dynamic collections render through `List`/`ForEach`, which need a stable
identity per row (`Identifiable`, or an explicit `id:` key path) to animate
insertions/removals correctly:

```swift
struct Task: Identifiable {
    let id = UUID()
    var title: String
    var isDone: Bool
}

struct TaskListView: View {
    @State private var tasks: [Task] = [
        Task(title: "Write SwiftUI notes", isDone: false),
        Task(title: "Review PR", isDone: true)
    ]

    var body: some View {
        List {
            ForEach($tasks) { $task in
                Toggle(task.title, isOn: $task.isDone)
            }
        }
    }
}
```

`ForEach($tasks)` iterates *bindings* into each array element (available
because `tasks` is `@State` and `Task` is `Identifiable`), which is what
lets each row's `Toggle` mutate its own element directly, in place, inside
the array.

## Swift-specific traps

- **`@State` should be `private`.** A `@State` property with external
  access invites another view to read (or worse, expect to write) storage
  that's meant to be owned entirely by the declaring view — SwiftUI won't
  stop you, but it breaks the ownership model the framework assumes.
- **`@Observable` classes are reference types** — passing one to a child
  view (as `CartView` does above, with a plain `let cart: CartModel`) means
  every view holding it shares the *same* instance; there's no copy, unlike
  passing a `struct`.
- **Recomputing `body` is not free** — a `body` that does expensive work
  (not just describing views) runs on every state change that invalidates
  it; keep `body` cheap and push real computation into the model layer.
- **Forgetting `Identifiable` (or an explicit `id:`)** on `ForEach` data
  forces SwiftUI to fall back to positional identity, which produces subtle
  bugs when rows are inserted, removed, or reordered — always give dynamic
  list content real, stable identity.

## Cheat sheet

| Need | Tool |
|------|------|
| Local, view-owned mutable state | `@State private var` |
| Pass mutable access to a child | `@Binding` + `$` projection |
| Shared, reference-type state | `@Observable` class |
| Style/layout a view | Chained modifiers (order matters) |
| Render a dynamic collection | `List`/`ForEach` + `Identifiable` |

## How It Actually Works

- **A `View`'s `body` is not "the UI" — it's a lightweight, disposable value
  describing what the UI should look like.** SwiftUI treats your `struct MyView:
  View`'s `body` computed property as a pure function of its inputs (properties,
  `@State`, `@Binding`, environment) — the framework calls it repeatedly
  (potentially many times per second) and never mutates or reuses the returned
  view value directly; it's recomputed fresh each time and diffed.
- **Diffing works by view *identity*, not by reference equality.** SwiftUI
  assigns each view in your hierarchy an implicit identity based on its type and
  position in the view tree (or an explicit `.id(...)`/`ForEach` id you
  provide). When `body` re-runs, SwiftUI walks the old and new view trees and
  compares node-by-node at matching identities: if a node's *type* changed, the
  old view (and any attached state) is torn down and a fresh one created; if
  the type is the same, SwiftUI diffs the view's stored properties and only
  updates the underlying platform view (UIView/NSView, or its own render tree)
  for properties that actually changed — this selective re-rendering is the
  entire performance model, and it's why `ForEach` needs stable, meaningful
  `id`s: an unstable id makes SwiftUI think every element is brand new on every
  update, discarding state and animating incorrectly.
- **`@State` storage doesn't live in your view struct at all** (your view is
  re-created constantly, so it can't). The compiler-inserted property-wrapper
  storage for `@State` is actually persisted by the SwiftUI runtime in a
  separate, identity-keyed storage table outside your view value — this is why
  `@State` survives across `body` re-evaluations even though the `View` struct
  itself is a fresh value every time, and why `@State` is documented as "only
  for a view's own private, local state," not something you should initialize
  from an outside parameter (the framework only initializes it once per
  identity, ignoring later external re-initialization attempts).
- **Every state change triggers invalidation, not immediate re-render**:
  mutating an `@State`/`@ObservedObject`-published property marks the
  associated view identity "dirty" and schedules a re-render on the next run
  loop tick, batching multiple state changes in the same tick into a single
  `body` re-evaluation rather than one per mutation.

## Exercise

Sketch a SwiftUI `LoginView` with `@State private var username` and
`@State private var password`, two `TextField`/`SecureField` rows bound to
them, and a `Button("Log In")` disabled (via `.disabled(...)`) whenever
either field is empty. Extract the disabled-check into a computed property
`isFormValid: Bool` on the view. Then convert the same screen to use an
`@Observable class LoginFormModel` holding both fields and the
`isFormValid` logic instead, and note in a comment which version you'd
prefer for a form this simple, and why.
