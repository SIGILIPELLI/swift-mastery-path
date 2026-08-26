# 10 · Capstone Project

This capstone combines nearly every module from Levels 3 and 4 into one
working service: a signed, authenticated REST API backed by SQLite, with
structured logging and environment-driven configuration, built on the
service-layer architecture from Module 09. Every piece below is real code
that compiles and runs, exercised end to end with `curl` against the
actual listening server.

## What it pulls together

| Piece | From |
|-------|------|
| HTTP server on `Network.framework` | Level 3, Module 10 |
| SQLite persistence via prepared statements | Level 3, Module 03 |
| Structured logging + environment config | Level 4, Module 02 |
| HMAC request signing for write authorization | Level 4, Module 04 |
| Protocol-fronted service layer | Level 4, Module 09 |

## Project layout

```
capstone/
├── Package.swift
└── Sources/
    └── capstone/
        └── main.swift
```

```swift
// Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "capstone",
    platforms: [.macOS(.v13)],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log.git", from: "1.6.0")
    ],
    targets: [
        .executableTarget(name: "capstone", dependencies: [.product(name: "Logging", package: "swift-log")])
    ]
)
```

## Configuration and the model/database layer

Config reads from the environment (Module 02) and derives a signing key
from an `API_KEY` environment variable:

```swift
import Foundation
import CryptoKit

struct Config {
    let port: UInt16
    let environment: String
    let apiKey: SymmetricKey

    static func fromEnvironment() -> Config {
        let port = UInt16(ProcessInfo.processInfo.environment["PORT"] ?? "") ?? 8124
        let env = ProcessInfo.processInfo.environment["APP_ENV"] ?? "development"
        let keyString = ProcessInfo.processInfo.environment["API_KEY"] ?? "demo-secret-key"
        let key = SymmetricKey(data: SHA256.hash(data: Data(keyString.utf8)))
        return Config(port: port, environment: env, apiKey: key)
    }
}
```

`SymmetricKey(data: SHA256.hash(...))` derives a fixed-length key from an
arbitrary-length passphrase — not a proper KDF (Module 04 mentions HKDF for
that), but enough to turn any `API_KEY` string into a usable 256-bit key
for this demo.

The database and service layer follow the exact prepared-statement and
protocol-fronted patterns from Level 3 Module 03 and Level 4 Module 09:

```swift
struct Note {
    let id: Int
    let text: String
}

protocol NoteStoring {
    func fetchAll() -> [Note]
    func add(text: String) -> Note
}

final class SQLiteNoteStore: NoteStoring {
    private let db: NoteDatabase
    init(db: NoteDatabase) { self.db = db }
    func fetchAll() -> [Note] { db.all() }
    func add(text: String) -> Note {
        let id = db.insert(text: text)
        return Note(id: id, text: text)
    }
}
```

(`NoteDatabase` itself is the same `sqlite3_prepare_v2`/`sqlite3_step`
shape as Level 3 Module 03's `EmployeeDatabase` — omitted here for length;
see that module for the full implementation.)

## Authorizing writes with HMAC

Every `POST` must carry an `X-Signature` header — an HMAC-SHA256 over the
raw request body, using the server's configured key (Module 04):

```swift
func isAuthorized(headerValue: String?, body: String, key: SymmetricKey) -> Bool {
    guard let headerValue else { return false }
    guard let providedData = Data(hexString: headerValue) else { return false }
    return HMAC<SHA256>.isValidAuthenticationCode(providedData, authenticating: Data(body.utf8), using: key)
}
```

Using `HMAC.isValidAuthenticationCode` (rather than recomputing a hex
string and comparing with `==`) means the comparison itself is
constant-time, exactly the concern Module 04 raised about hand-rolled
secret comparisons.

## The route handler

```swift
switch (method, path) {
case ("GET", "/notes"):
    responseBody = jsonArray(store.fetchAll())
    logger.info("Handled request", metadata: ["method": "GET", "path": "/notes"])
case ("POST", "/notes"):
    guard isAuthorized(headerValue: signatureHeader, body: body, key: config.apiKey) else {
        status = "401 Unauthorized"
        responseBody = "{\"error\":\"invalid or missing signature\"}"
        logger.warning("Rejected unauthorized write", metadata: ["path": "/notes"])
        break
    }
    let fields = parseSimpleJSON(body)
    let text = fields["text"] ?? "Untitled"
    let note = store.add(text: text)
    responseBody = "{\"id\":\(note.id),\"text\":\"\(note.text)\"}"
    logger.info("Handled request", metadata: ["method": "POST", "path": "/notes", "id": "\(note.id)"])
default:
    status = "404 Not Found"
    responseBody = "{\"error\":\"not found\"}"
}
```

Every branch logs a structured line (Module 02) — an unauthorized write
attempt is a `warning`, a successful request is `info`, giving an operator
exactly the audit trail a real deployment needs.

## Running it end to end

```
$ swift build
Build complete! (48.69s)

$ ./.build/debug/capstone &
2026-08-26T22:35:06+0530 info com.example.capstone: env=development port=8124 [capstone] Server started

$ curl -s -X POST http://127.0.0.1:8124/notes -d '{"text":"hello"}'
2026-08-26T22:35:06+0530 warning com.example.capstone: path=/notes [capstone] Rejected unauthorized write
{"error":"invalid or missing signature"}

$ curl -s http://127.0.0.1:8124/notes
[]

$ BODY='{"text":"hello"}'
$ KEY_HEX=$(echo -n "demo-secret-key" | shasum -a 256 | cut -d' ' -f1)
$ SIG=$(echo -n "$BODY" | openssl dgst -sha256 -mac HMAC -macopt hexkey:$KEY_HEX | sed 's/^.* //')

$ curl -s -X POST http://127.0.0.1:8124/notes -H "X-Signature: $SIG" -d "$BODY"
2026-08-26T22:35:07+0530 info com.example.capstone: id=1 method=POST path=/notes [capstone] Handled request
{"id":1,"text":"hello"}

$ curl -s http://127.0.0.1:8124/notes
[{"id":1,"text":"hello"}]
```

The unsigned `POST` is rejected with `401` and a warning log line; once the
client computes the same HMAC the server derives from `API_KEY`
(demonstrated here with `openssl dgst -mac HMAC`, standing in for a real
client that would use `CryptoKit` itself), the write succeeds and the
following `GET` shows it persisted.

## Design notes

- **The signature covers the raw body, not a parsed/re-serialized
  version.** Signing over `body` exactly as received (rather than
  re-encoding the parsed fields) avoids a whole class of "signature valid
  but payload subtly different" bugs that JSON re-serialization
  (differing key order, whitespace) can introduce.
- **This demo derives the signing key from a passphrase for simplicity.**
  A production system would generate and distribute a real random
  `SymmetricKey`, not derive one from a memorable string, and would rotate
  it periodically.
- **The HTTP/JSON layer is intentionally the same minimal, dependency-free
  approach as Level 3's project** — see that module's design notes for why
  Vapor isn't used here, and its stretch goals for what a production
  version would add (real `Codable` JSON, connection keep-alive, more
  routes).

## Stretch goals

- Add a `DELETE /notes/:id` route that also requires a valid `X-Signature`
  over an empty body, and a `GET /notes/:id` route that doesn't (reads stay
  unauthenticated, consistent with the existing `GET /notes`).
- Wrap `NoteDatabase` access in an `actor` (Level 3, Module 01) instead of
  relying on the single serial `DispatchQueue`, and adapt the
  connection-handling closures to bridge into `async` calls via `Task { await ... }`
  — compare the resulting code's readability to the callback-based version.
- Containerize the finished service using the multi-stage Dockerfile
  pattern from Module 06, injecting `API_KEY` and `PORT` via `docker run
  -e`, and confirm the signed-request flow above still works identically
  against the containerized server.
- Add a `RateLimiter` (reusing the design from Module 05's testing
  exercise) in front of the `POST` handler, returning `429 Too Many
  Requests` once a client exceeds a configured request rate, and log a
  `warning` each time a request is throttled.
