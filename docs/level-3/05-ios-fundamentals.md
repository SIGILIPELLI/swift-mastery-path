# 05 · iOS Fundamentals

This module covers the core building blocks of an iOS app: the app
lifecycle, view controllers, Auto Layout, and navigation. iOS code needs a
simulator or device to actually run.

!!! note "Environment note for this module"
    This machine has only the Xcode Command Line Tools (no full Xcode, no
    iOS SDK/simulator), so nothing in this module can be compiled or run
    here. The code below is hand-reviewed for correct, current UIKit/
    SwiftUI API usage — verify it in Xcode on a machine with the iOS SDK
    before shipping it.

## The app lifecycle

A modern iOS app (using the `SwiftUI` `App` protocol, the default since
Xcode 11) starts from a single entry point:

```swift
import SwiftUI

@main
struct TaskApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

Under the hood this still drives the same lifecycle events as the older
`UIApplicationDelegate` world — `scenePhase` in SwiftUI exposes them:

```swift
struct ContentView: View {
    @Environment(\.scenePhase) private var scenePhase

    var body: some View {
        Text("Hello, Task App")
            .onChange(of: scenePhase) { oldPhase, newPhase in
                switch newPhase {
                case .active: print("App became active")
                case .inactive: print("App became inactive")
                case .background: print("App entered background")
                @unknown default: break
                }
            }
    }
}
```

`.active` → `.inactive` → `.background` is the normal path when a user
backgrounds the app (say, by swiping to the home screen); the reverse path
runs on return.

## `UIViewController` basics (UIKit)

Projects that predate SwiftUI, or that need UIKit-only APIs, still use
`UIViewController` subclasses with a well-defined set of lifecycle hooks:

```swift
import UIKit

final class ProfileViewController: UIViewController {
    private let nameLabel = UILabel()

    override func viewDidLoad() {
        super.viewDidLoad()
        // one-time setup: build the view hierarchy
        view.backgroundColor = .systemBackground
        nameLabel.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(nameLabel)
        NSLayoutConstraint.activate([
            nameLabel.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            nameLabel.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        // runs every time this screen is about to become visible
        nameLabel.text = "Ada Lovelace"
    }
}
```

`viewDidLoad` runs once, when the view hierarchy is first created;
`viewWillAppear`/`viewDidAppear` run every time the screen becomes visible
again (e.g., after popping back from a pushed screen) — a common bug is
putting per-appearance setup (like refreshing data) in `viewDidLoad`, where
it only runs the first time.

## Auto Layout with anchors

The `NSLayoutConstraint` anchor API above is the standard programmatic Auto
Layout approach — each anchor call describes one relationship, and
`NSLayoutConstraint.activate` turns a batch of them on together (more
efficient than activating one at a time):

```swift
let button = UIButton(type: .system)
button.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(button)

NSLayoutConstraint.activate([
    button.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
    button.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
    button.bottomAnchor.constraint(equalTo: view.safeAreaLayoutGuide.bottomAnchor, constant: -24),
    button.heightAnchor.constraint(equalToConstant: 50)
])
```

`translatesAutoresizingMaskIntoConstraints = false` is required on every
view you plan to constrain manually — without it, UIKit's legacy
autoresizing-mask-to-constraints translation fights your explicit
constraints, a very common first bug.

## Navigation: `UINavigationController` and `NavigationStack`

UIKit pushes/pops view controllers on a navigation stack:

```swift
let detail = ProfileViewController()
navigationController?.pushViewController(detail, animated: true)
```

SwiftUI's equivalent is declarative, driven by a `NavigationStack` and a
value type describing the destination:

```swift
struct ContactListView: View {
    let names = ["Ada", "Grace", "Alan"]

    var body: some View {
        NavigationStack {
            List(names, id: \.self) { name in
                NavigationLink(name, value: name)
            }
            .navigationDestination(for: String.self) { name in
                Text("Profile for \(name)")
            }
            .navigationTitle("Contacts")
        }
    }
}
```

`navigationDestination(for:)` maps a value type to the view that should be
pushed when a `NavigationLink` carrying that type is tapped — this
decouples "what can be navigated to" from "where in the list it's
triggered from," which scales better than UIKit's segue-based navigation
for larger apps.

## Swift-specific traps

- **`@main` on the `App` struct is the entire entry point** — there's no
  `main.swift` or `AppDelegate.swift` to also define `main()` unless you
  opt into the UIKit app-delegate lifecycle explicitly via `@UIApplicationDelegateAdaptor`.
- **Forgetting `translatesAutoresizingMaskIntoConstraints = false`** is the
  single most common Auto Layout bug for programmatic UIKit layouts —
  constraints silently conflict with the auto-generated ones.
- **`viewDidLoad` vs `viewWillAppear`**: data that can go stale (network
  results, user defaults) belongs in `viewWillAppear` or later, not
  `viewDidLoad`, which only fires once per view controller instance.
- **SwiftUI's `List(names, id: \.self)`** requires `names`' elements to be
  genuinely unique — duplicate strings produce undefined row identity and
  visual glitches; prefer an `Identifiable` model type with a real `id`
  once data isn't guaranteed-unique strings.

## Cheat sheet

| Concept | UIKit | SwiftUI |
|---------|-------|---------|
| Entry point | `AppDelegate` + `SceneDelegate` | `@main struct MyApp: App` |
| Screen | `UIViewController` | `View` (a struct) |
| One-time setup | `viewDidLoad()` | `.onAppear` (careful: can rerun) / `init` |
| Layout | `NSLayoutConstraint` anchors | Stacks (`VStack`/`HStack`) + modifiers |
| Push a screen | `navigationController?.pushViewController` | `NavigationLink` + `navigationDestination` |

## Exercise

Sketch (don't need to run) a SwiftUI `TaskListView` backed by an
`@Observable class TaskStore` holding `[Task]` (where `Task` has `id`,
`title`, `isDone`). Show: a `List` iterating the store's tasks with a
`Toggle` bound to each `isDone`, a toolbar `+` button that appends a new
`Task`, and a `NavigationStack` wrapping the whole thing with the title
"Tasks". Note in a comment which lifecycle event you'd use to load
persisted tasks when the view first appears.
