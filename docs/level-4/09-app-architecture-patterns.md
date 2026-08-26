# 09 · App Architecture Patterns

Beyond individual design patterns (Level 3, Module 04), a whole app needs a
consistent shape: where business logic lives, how views get their data, and
how navigation is decided. This module builds MVVM and Coordinator — the
two patterns most Swift apps reach for — as plain Swift, so you can see and
run the layering without any UI framework in the way.

## Layering: model, service, view model

**Model** — plain data, no behavior beyond what describes the data itself:

```swift
struct Article {
    let id: Int
    let title: String
    let isRead: Bool
}
```

**Service** — a protocol-fronted boundary to the outside world (network,
database), exactly the dependency-injection shape from Level 3's testing
module:

```swift
protocol ArticleFetching {
    func fetchArticles() async -> [Article]
}

struct LiveArticleService: ArticleFetching {
    func fetchArticles() async -> [Article] {
        [
            Article(id: 1, title: "Swift 6 Concurrency", isRead: false),
            Article(id: 2, title: "MVVM in Practice", isRead: true)
        ]
    }
}
```

## MVVM: the view model as the only thing a view talks to

The view model depends on the service (not a concrete implementation),
exposes only what a view needs to render, and does all the
data-transformation work a view shouldn't have to:

```swift
import Combine

@MainActor
final class ArticleListViewModel: ObservableObject {
    @Published private(set) var titles: [String] = []
    @Published private(set) var unreadCount: Int = 0

    private let service: ArticleFetching

    init(service: ArticleFetching) {
        self.service = service
    }

    func load() async {
        let articles = await service.fetchArticles()
        titles = articles.map { $0.title }
        unreadCount = articles.filter { !$0.isRead }.count
    }
}
```

`@MainActor` on the whole class guarantees every property mutation and
method call happens on the main thread — required because `@Published`
properties drive UI updates, and UI must only ever be touched from the
main thread. A SwiftUI view would hold this as `@StateObject` or (using
`@Observable`, Module 03) a plain reference, and read `titles`/`unreadCount`
directly in its `body` — the view never touches `ArticleFetching` at all.

## Coordinator: navigation decisions live outside the view

Without a coordinator, navigation logic tends to spread across many views
(`NavigationLink` destinations, `pushViewController` calls scattered
everywhere), making the overall flow hard to see or change in one place. A
coordinator centralizes it:

```swift
protocol Route: Equatable {}
enum AppRoute: Route {
    case articleList
    case articleDetail(id: Int)
    case settings
}

final class Coordinator {
    private(set) var stack: [AppRoute] = [.articleList]

    func push(_ route: AppRoute) {
        stack.append(route)
        print("Navigated to \(route), stack depth: \(stack.count)")
    }

    func pop() {
        guard stack.count > 1 else { return }
        let popped = stack.removeLast()
        print("Popped \(popped), stack depth: \(stack.count)")
    }
}
```

Views ask the coordinator to navigate (`coordinator.push(.settings)`)
rather than constructing and presenting the next screen themselves — this
is what lets you change "tapping this row goes to a detail screen" to
"...goes to a paywall first" in one place, without touching the view that
triggered it.

## Full run

```swift
@MainActor
func runDemo() async {
    let viewModel = ArticleListViewModel(service: LiveArticleService())
    await viewModel.load()
    print("Titles:", viewModel.titles)
    print("Unread count:", viewModel.unreadCount)

    let coordinator = Coordinator()
    coordinator.push(.articleDetail(id: 1))
    coordinator.push(.settings)
    coordinator.pop()
}

await runDemo()
```

Output:

```
Titles: ["Swift 6 Concurrency", "MVVM in Practice"]
Unread count: 1
Navigated to articleDetail(id: 1), stack depth: 2
Navigated to settings, stack depth: 3
Popped settings, stack depth: 2
```

## Where this fits with SwiftUI specifically

SwiftUI's `NavigationStack` (Module 03) already gives you a
value-describing-a-destination model natively via
`navigationDestination(for:)` — many SwiftUI codebases fold the
Coordinator's job into an `@Observable` router class holding a `[AppRoute]`
path bound directly to `NavigationStack(path:)`, rather than a
UIKit-flavored `Coordinator` object pushing view controllers. The
underlying idea — navigation state lives in one testable place, not
scattered across views — is identical either way.

## Swift-specific traps

- **`@MainActor` on the view model, but not on the service**, is
  deliberate: the service does the actual `async` work (which might run
  off the main thread, e.g. inside a URLSession task), and only the final
  hop back into the view model's `@MainActor`-isolated methods needs to be
  on the main thread. Marking the service `@MainActor` too would force
  network/disk I/O onto the main thread unnecessarily.
- **A view model depending on a concrete service type instead of a
  protocol** silently reintroduces the untestability Level 3's testing
  module warned about — the whole point of `ArticleFetching` is that tests
  can substitute a fake without touching `ArticleListViewModel` at all.
- **Coordinators can accumulate into a "God object"** if every screen's
  entire flow logic funnels through one giant `Coordinator` class — for
  larger apps, splitting into per-feature coordinators (an
  `ArticleCoordinator`, a `SettingsCoordinator`) composed by a root
  coordinator scales better than one flat enum of every route in the app.
- **`Route: Equatable` matters for `NavigationStack(path:)` binding** in
  the SwiftUI-native version — the path array's elements need identity
  comparison for SwiftUI to know what changed.

## Cheat sheet

| Layer | Responsibility | Depends on |
|-------|----------------|------------|
| Model | Plain data | Nothing |
| Service | I/O, behind a protocol | Nothing app-specific |
| View model | Presentation state + transformation | A service protocol |
| View | Rendering + user input | A view model |
| Coordinator/Router | Navigation decisions | Route values, not views directly |

## Exercise

Add a `SettingsViewModel` with a `@Published var isDarkModeEnabled: Bool`
and a `toggleDarkMode()` method, following the same `@MainActor` +
protocol-backed-service shape as `ArticleListViewModel` (even if the
"service" here is trivial, like a `UserDefaultsSettingsStore` protocol with
a fake test double). Extend `Coordinator` with a `func
presentSettings()` that pushes `.settings`, then write a small test-style
function (no `XCTest` needed, plain assertions with `assert(...)` are
fine) verifying that after `presentSettings()` the stack's last element is
`.settings`.
