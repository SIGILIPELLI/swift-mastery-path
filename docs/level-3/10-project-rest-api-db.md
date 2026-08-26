# 10 · Project — REST API + Database Service

This capstone for Level 3 combines Module 01 (concurrency), Module 03
(databases), and Module 08's copy-on-write value types into one small but
complete, dependency-free REST service: a JSON API backed by SQLite,
listening on a real TCP socket.

!!! note "Why not Vapor here"
    Module 02 covers Vapor's actual API in detail, but Vapor can't compile
    in this CLT-only environment (see that module's note — its TLS
    dependency needs full Xcode's C++ headers). This project instead uses
    Apple's system `Network` framework directly, which ships with the OS
    and has no such dependency, so the whole thing genuinely builds and
    runs here. The HTTP parsing below is intentionally minimal — a
    production service should use Vapor, Hummingbird, or another real
    framework rather than hand-rolled HTTP.

## Project layout

```
restapi/
├── Package.swift
└── Sources/
    └── restapi/
        └── main.swift
```

```swift
// Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "restapi",
    platforms: [.macOS(.v13)],
    targets: [
        .executableTarget(name: "restapi")
    ]
)
```

## The database layer

Reusing the prepared-statement patterns from Module 03, wrapped behind a
small type so the server code above it never touches SQLite directly:

```swift
import SQLite3

final class EmployeeDatabase {
    private var db: OpaquePointer?

    init(path: String) {
        sqlite3_open(path, &db)
        exec("""
        CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            role TEXT NOT NULL
        );
        """)
    }

    deinit { sqlite3_close(db) }

    private func exec(_ sql: String) {
        sqlite3_exec(db, sql, nil, nil, nil)
    }

    func all() -> [(id: Int, name: String, role: String)] {
        let sql = "SELECT id, name, role FROM employees ORDER BY id;"
        var statement: OpaquePointer?
        sqlite3_prepare_v2(db, sql, -1, &statement, nil)
        defer { sqlite3_finalize(statement) }

        var results: [(id: Int, name: String, role: String)] = []
        while sqlite3_step(statement) == SQLITE_ROW {
            let id = Int(sqlite3_column_int(statement, 0))
            let name = String(cString: sqlite3_column_text(statement, 1))
            let role = String(cString: sqlite3_column_text(statement, 2))
            results.append((id, name, role))
        }
        return results
    }

    func insert(name: String, role: String) -> Int {
        let sql = "INSERT INTO employees (name, role) VALUES (?, ?);"
        var statement: OpaquePointer?
        sqlite3_prepare_v2(db, sql, -1, &statement, nil)
        defer { sqlite3_finalize(statement) }

        let transient = unsafeBitCast(-1, to: sqlite3_destructor_type.self)
        sqlite3_bind_text(statement, 1, name, -1, transient)
        sqlite3_bind_text(statement, 2, role, -1, transient)
        sqlite3_step(statement)
        return Int(sqlite3_last_insert_rowid(db))
    }
}
```

## A tiny hand-rolled JSON encoder

No `Codable` dependency needed for two known-shape rows — this is
intentionally minimal, escaping only the one character (`"`) that would
otherwise break the output:

```swift
func jsonArray(_ rows: [(id: Int, name: String, role: String)]) -> String {
    let items = rows.map { row -> String in
        let name = row.name.replacingOccurrences(of: "\"", with: "\\\"")
        let role = row.role.replacingOccurrences(of: "\"", with: "\\\"")
        return "{\"id\":\(row.id),\"name\":\"\(name)\",\"role\":\"\(role)\"}"
    }
    return "[" + items.joined(separator: ",") + "]"
}
```

## The HTTP layer on `Network.framework`

`NWListener` accepts TCP connections; each connection is handed a minimal
HTTP/1.1 request parser and responder. The class is marked
`@unchecked Sendable` because all of its mutable state (the SQLite handle)
is only ever touched from the single serial `queue` the listener and every
connection callback run on:

```swift
import Network
import Foundation

final class HTTPServer: @unchecked Sendable {
    private let listener: NWListener
    private let db: EmployeeDatabase
    private let queue = DispatchQueue(label: "http-server")

    init(port: UInt16, db: EmployeeDatabase) throws {
        self.db = db
        listener = try NWListener(using: .tcp, on: NWEndpoint.Port(rawValue: port)!)
    }

    func start() {
        listener.newConnectionHandler = { [weak self] connection in
            self?.handle(connection)
        }
        listener.start(queue: queue)
    }

    func stop() { listener.cancel() }

    private func handle(_ connection: NWConnection) {
        connection.start(queue: queue)
        receive(on: connection, buffer: Data())
    }

    private func receive(on connection: NWConnection, buffer: Data) {
        connection.receive(minimumIncompleteLength: 1, maximumLength: 65536) { [weak self] data, _, isComplete, error in
            guard let self else { return }
            var buffer = buffer
            if let data, !data.isEmpty { buffer.append(data) }

            if let requestText = String(data: buffer, encoding: .utf8), requestText.contains("\r\n\r\n") {
                self.respond(to: requestText, on: connection)
            } else if isComplete || error != nil {
                connection.cancel()
            } else {
                self.receive(on: connection, buffer: buffer)
            }
        }
    }

    private func respond(to requestText: String, on connection: NWConnection) {
        let requestLine = requestText.components(separatedBy: "\r\n").first ?? ""
        let parts = requestLine.split(separator: " ")
        let method = parts.count > 0 ? String(parts[0]) : ""
        let path = parts.count > 1 ? String(parts[1]) : ""

        var status = "200 OK"
        var body = "{}"

        switch (method, path) {
        case ("GET", "/employees"):
            body = jsonArray(db.all())
        case ("POST", "/employees"):
            let bodyStart = requestText.range(of: "\r\n\r\n").map { requestText[$0.upperBound...] } ?? ""
            let fields = parseSimpleJSON(String(bodyStart))
            let name = fields["name"] ?? "Unknown"
            let role = fields["role"] ?? "Unknown"
            let id = db.insert(name: name, role: role)
            body = "{\"id\":\(id),\"name\":\"\(name)\",\"role\":\"\(role)\"}"
        default:
            status = "404 Not Found"
            body = "{\"error\":\"not found\"}"
        }

        let response = "HTTP/1.1 \(status)\r\nContent-Type: application/json\r\nContent-Length: \(body.utf8.count)\r\nConnection: close\r\n\r\n\(body)"
        connection.send(content: response.data(using: .utf8), completion: .contentProcessed { _ in
            connection.cancel()
        })
    }

    private func parseSimpleJSON(_ text: String) -> [String: String] {
        var result: [String: String] = [:]
        let trimmed = text.trimmingCharacters(in: .whitespacesAndNewlines)
            .trimmingCharacters(in: CharacterSet(charactersIn: "{}"))
        for pair in trimmed.components(separatedBy: ",") {
            let kv = pair.components(separatedBy: ":")
            guard kv.count == 2 else { continue }
            let key = kv[0].trimmingCharacters(in: CharacterSet(charactersIn: " \""))
            let value = kv[1].trimmingCharacters(in: CharacterSet(charactersIn: " \""))
            result[key] = value
        }
        return result
    }
}
```

## Entry point

```swift
let dbPath = "/tmp/restapi_demo.sqlite"
try? FileManager.default.removeItem(atPath: dbPath)
let database = EmployeeDatabase(path: dbPath)
_ = database.insert(name: "Ada", role: "Engineer")

let server = try HTTPServer(port: 8123, db: database)
server.start()

print("Server listening on http://127.0.0.1:8123")
RunLoop.main.run(until: Date().addingTimeInterval(3))
server.stop()
```

## Running it

```
$ swift build
Build complete! (3.28s)

$ ./.build/debug/restapi &
Server listening on http://127.0.0.1:8123

$ curl -s http://127.0.0.1:8123/employees
[{"id":1,"name":"Ada","role":"Engineer"}]

$ curl -s -X POST http://127.0.0.1:8123/employees -d '{"name":"Grace","role":"Engineer"}'
{"id":2,"name":"Grace","role":"Engineer"}

$ curl -s http://127.0.0.1:8123/employees
[{"id":1,"name":"Ada","role":"Engineer"},{"id":2,"name":"Grace","role":"Engineer"}]
```

All three requests were run against the real listening server on this
machine — the SQLite-backed `POST` genuinely persists the new row, which
the following `GET` reflects.

## Design notes

- **Serial queue instead of an actor.** `NWListener`/`NWConnection`
  callbacks predate Swift concurrency and hand you closures on a
  `DispatchQueue` you choose — using a single serial queue for the listener
  and every connection gives the same "only one thing touches the database
  at a time" guarantee an actor would, without mixing callback-based
  Network.framework APIs with `async`/`await`.
- **The JSON parsing is deliberately naive.** It splits on `,` and `:`,
  which breaks on nested objects, arrays, or commas inside string values.
  A real project would use `JSONDecoder` against a `Codable` struct — this
  project avoids `Codable` only to keep the whole example dependency-free
  and inspectable end to end.
- **No connection keep-alive.** Every response includes `Connection:
  close` and cancels the connection after sending — simple and correct for
  a demo, but a production server keeps connections open (HTTP/1.1
  keep-alive) for efficiency.

## Stretch goals

- Add a `GET /employees/:id` route (parse the id out of the path) and a
  `DELETE /employees/:id` route, returning a 404 JSON body for a missing
  id.
- Replace the naive JSON parser with `JSONDecoder`/`JSONEncoder` against
  `Codable` request/response structs, and measure whether that changes the
  server's behavior for a body containing a comma inside a string value
  (e.g. `{"name":"Smith, Jr.","role":"Engineer"}`) — it doesn't handle that
  correctly today.
- Swap the single serial `DispatchQueue` for an `actor EmployeeStore`
  wrapping the database calls, and adapt the connection-handling closures
  to `Task { await ... }` bridges, to compare the two concurrency styles
  covered in Module 01.
- Once a full Xcode toolchain is available, port this project onto Vapor
  using the routes and middleware shown in Module 02, and compare how much
  of the hand-rolled HTTP/JSON code Vapor eliminates.
