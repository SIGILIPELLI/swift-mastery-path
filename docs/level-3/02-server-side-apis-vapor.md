# 02 · Building Server-Side APIs (Vapor)

Vapor is Swift's most widely used server-side web framework: a routing
layer, a `Codable`-based request/response pipeline, and an async runtime
built on SwiftNIO. This module builds a small JSON API and explains the
pieces you'll reuse in every Vapor project.

!!! note "Environment note for this module"
    Vapor pulls in SwiftNIO's TLS support (`swift-nio-ssl`, which vendors
    BoringSSL as C++), and compiling that requires a full Xcode toolchain's
    C++ standard library headers. This machine has only the Xcode Command
    Line Tools, so `swift build` fails at `CNIOBoringSSL` with `'memory'
    file not found` before it ever reaches Vapor's own Swift code — the
    same limitation noted for XCTest elsewhere in this course. The code
    below is real, idiomatic Vapor 4 API, hand-reviewed for correctness
    against Vapor's documented interfaces; verify it compiles and runs with
    `swift run` on a machine with full Xcode installed.

## Setting up a Vapor package

A Vapor project is a normal SPM executable target with one dependency:

```swift
// Package.swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "TodoAPI",
    platforms: [.macOS(.v13)],
    dependencies: [
        .package(url: "https://github.com/vapor/vapor.git", from: "4.99.0")
    ],
    targets: [
        .executableTarget(
            name: "TodoAPI",
            dependencies: [.product(name: "Vapor", package: "vapor")]
        )
    ]
)
```

## A `Content` model

Vapor decodes and encodes JSON through types conforming to `Content`
(itself `Codable` plus response/request helpers):

```swift
import Vapor

struct Todo: Content {
    let id: Int
    var title: String
    var done: Bool
}
```

## Storing state safely: an actor-backed store

Route closures run concurrently, so shared mutable state needs the same
actor protection as any other concurrent Swift code (see Module 01):

```swift
actor TodoStore {
    private var todos: [Todo] = [Todo(id: 1, title: "Learn Vapor", done: false)]
    private var nextID = 2

    func all() -> [Todo] { todos }

    func add(title: String) -> Todo {
        let todo = Todo(id: nextID, title: title, done: false)
        nextID += 1
        todos.append(todo)
        return todo
    }

    func toggle(id: Int) -> Todo? {
        guard let index = todos.firstIndex(where: { $0.id == id }) else { return nil }
        todos[index].done.toggle()
        return todos[index]
    }
}
```

## Routing

`Application` exposes `get`, `post`, `put`, `delete` for registering routes.
Handlers are `async throws` closures that return anything conforming to
`ResponseEncodable` — `Content` types qualify automatically:

```swift
import Vapor

let store = TodoStore()
let app = try await Application.make(.detect())

app.get("todos") { req async throws -> [Todo] in
    await store.all()
}

app.post("todos") { req async throws -> Todo in
    struct Input: Content { let title: String }
    let input = try req.content.decode(Input.self)
    return await store.add(title: input.title)
}

app.put("todos", ":id") { req async throws -> Todo in
    guard let id = req.parameters.get("id", as: Int.self) else {
        throw Abort(.badRequest, reason: "id must be an integer")
    }
    guard let updated = await store.toggle(id: id) else {
        throw Abort(.notFound)
    }
    return updated
}

try await app.execute()
```

`req.parameters.get("id", as: Int.self)` pulls the `:id` path component out
of `PUT /todos/:id` and converts it; `Abort(.notFound)` short-circuits the
handler with an HTTP 404 and a JSON error body Vapor generates for you.

## Trying it (on a machine with full Xcode)

```
$ swift run
Server starting on http://127.0.0.1:8080

$ curl http://127.0.0.1:8080/todos
[{"id":1,"title":"Learn Vapor","done":false}]

$ curl -X POST http://127.0.0.1:8080/todos -d '{"title":"Ship the API"}' -H "Content-Type: application/json"
{"id":2,"title":"Ship the API","done":false}

$ curl -X PUT http://127.0.0.1:8080/todos/2
{"id":2,"title":"Ship the API","done":true}
```

## Middleware

Middleware wraps every request/response pair — logging, auth headers, CORS.
Vapor applies it via `app.middleware.use(...)`:

```swift
struct RequestTimer: AsyncMiddleware {
    func respond(to request: Request, chainingTo next: AsyncResponder) async throws -> Response {
        let start = Date()
        let response = try await next.respond(to: request)
        let elapsed = Date().timeIntervalSince(start)
        request.logger.info("Handled in \(elapsed)s")
        return response
    }
}

app.middleware.use(RequestTimer())
```

Middleware order matters: each one calls `next.respond(to:)` to hand off to
the next link in the chain (or the final route handler), so code before that
call runs on the way in and code after it runs on the way out.

## Route grouping

Related routes are usually grouped under a shared path prefix and shared
middleware rather than repeated per-route:

```swift
let api = app.grouped("api", "v1")
let todos = api.grouped(RequestTimer())
todos.get("todos") { req async throws -> [Todo] in await store.all() }
```

## Swift-specific traps

- **`Application.make` vs `Application(_:)`.** Modern Vapor (4.9+) uses the
  `async` factory `Application.make(_:)` so setup can itself be async;
  mixing the older synchronous initializer pattern with `async` route
  handlers from a tutorial written for an earlier Vapor version is a common
  source of confusion.
- **Forgetting `try await app.asyncShutdown()`** (or `app.execute()`, which
  runs until shutdown) leaves resources like the event loop group running —
  always pair app creation with an explicit run/shutdown path.
- **`Content` requires `Codable`, but not every `Codable` type is a good
  API shape.** Don't reuse database/Fluent models directly as `Content`
  types for public endpoints — a `DTO` struct decouples your wire format
  from your storage format so one can change without breaking the other.
- **Route parameter names must match between registration and lookup.**
  `app.put("todos", ":id")` and `req.parameters.get("id", ...)` have to use
  the identical string — a typo compiles fine and fails silently at
  runtime with a nil parameter.

## Cheat sheet

| Concept | API |
|---------|-----|
| Define a route | `app.get("path") { req async throws -> T in ... }` |
| Path parameter | `app.get("todos", ":id")` + `req.parameters.get("id", as: Int.self)` |
| Decode request body | `try req.content.decode(MyInput.self)` |
| Return an error | `throw Abort(.notFound)` |
| Add middleware | `app.middleware.use(MyMiddleware())` |
| Group routes | `app.grouped("api", "v1")` |

## Exercise

Extend the `TodoStore`/routing example with a `DELETE /todos/:id` route that
removes a todo by id, returning `HTTPStatus.noContent` on success and
`Abort(.notFound)` if the id doesn't exist. Add a `GET /todos/:id` route
that returns a single `Todo` the same way. Sketch out the `curl` commands
you'd use to exercise all five routes (`GET` list, `GET` one, `POST`, `PUT`,
`DELETE`) end to end.
