---
description: "Databases — Server-side and even command-line Swift programs frequently need to persist structured data. This module uses SQLite through Swift's bundled…"
---

# 03 · Databases

Server-side and even command-line Swift programs frequently need to persist
structured data. This module uses SQLite through Swift's bundled `SQLite3`
module — no external package needed — to cover the core relational
operations, then discusses where higher-level tools like Fluent (used by
Vapor) or `GRDB.swift` fit in.

## Opening a database

`SQLite3` is a thin C API wrapper that ships with the Swift toolchain. You
open a database file (or `:memory:` for an ephemeral one) into an
`OpaquePointer`:

```swift
import SQLite3
import Foundation

var db: OpaquePointer?
let dbPath = "/tmp/company.sqlite"
try? FileManager.default.removeItem(atPath: dbPath)   // start clean for this demo

guard sqlite3_open(dbPath, &db) == SQLITE_OK else {
    fatalError("Could not open database")
}
defer { sqlite3_close(db) }
```

## Running statements without parameters

`sqlite3_exec` runs SQL that doesn't need bound values — schema
definitions, one-off statements:

```swift
func exec(_ sql: String) {
    var errMsg: UnsafeMutablePointer<Int8>?
    if sqlite3_exec(db, sql, nil, nil, &errMsg) != SQLITE_OK {
        let message = errMsg.map { String(cString: $0) } ?? "unknown error"
        fatalError("SQL error: \(message)")
    }
}

exec("""
CREATE TABLE employees (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    role TEXT NOT NULL
);
""")
```

## Prepared statements: inserting safely

Never build SQL by interpolating strings — it's the classic SQL-injection
vector. Prepared statements with `?` placeholders bind values separately
from the query text:

```swift
func insert(name: String, role: String) {
    let sql = "INSERT INTO employees (name, role) VALUES (?, ?);"
    var statement: OpaquePointer?
    guard sqlite3_prepare_v2(db, sql, -1, &statement, nil) == SQLITE_OK else {
        fatalError("Prepare failed")
    }
    defer { sqlite3_finalize(statement) }   // always release the statement

    // SQLITE_TRANSIENT tells SQLite to copy the string, since Swift owns the buffer
    sqlite3_bind_text(statement, 1, name, -1, unsafeBitCast(-1, to: sqlite3_destructor_type.self))
    sqlite3_bind_text(statement, 2, role, -1, unsafeBitCast(-1, to: sqlite3_destructor_type.self))

    guard sqlite3_step(statement) == SQLITE_DONE else {
        fatalError("Insert failed")
    }
}

insert(name: "Ada", role: "Engineer")
insert(name: "Grace", role: "Engineer")
insert(name: "Alan", role: "Researcher")
```

`sqlite3_bind_text`'s last argument is a destructor telling SQLite how to
manage the string's memory. `SQLITE_TRANSIENT` (represented here as the
bit-cast of `-1`) tells it to make its own copy immediately, which is the
safe default when the Swift string might not outlive the call.

## Querying rows

`sqlite3_step` advances a prepared `SELECT` one row at a time, returning
`SQLITE_ROW` while there's more data and `SQLITE_DONE` when finished:

```swift
struct Employee {
    let id: Int
    let name: String
    let role: String
}

func fetchAll() -> [Employee] {
    let sql = "SELECT id, name, role FROM employees ORDER BY id;"
    var statement: OpaquePointer?
    guard sqlite3_prepare_v2(db, sql, -1, &statement, nil) == SQLITE_OK else {
        fatalError("Prepare failed")
    }
    defer { sqlite3_finalize(statement) }

    var results: [Employee] = []
    while sqlite3_step(statement) == SQLITE_ROW {
        let id = Int(sqlite3_column_int(statement, 0))
        let name = String(cString: sqlite3_column_text(statement, 1))
        let role = String(cString: sqlite3_column_text(statement, 2))
        results.append(Employee(id: id, name: name, role: role))
    }
    return results
}

for employee in fetchAll() {
    print("\(employee.id): \(employee.name) - \(employee.role)")
}

let engineers = fetchAll().filter { $0.role == "Engineer" }
print("Engineer count:", engineers.count)
```

Running this end to end (`swift databases.swift`):

```
1: Ada - Engineer
2: Grace - Engineer
3: Alan - Researcher
Engineer count: 2
```

Column indices in `sqlite3_column_*` are zero-based and follow the order of
columns in the `SELECT` list — `id` is 0, `name` is 1, `role` is 2 above.

## Where a higher-level library helps

Raw `SQLite3` is verbose and easy to get wrong (mismatched bind indices, a
forgotten `sqlite3_finalize`, no compile-time schema checking). In real
projects you'll usually reach for:

| Library | Best for |
|---------|----------|
| Fluent (Vapor) | Server-side apps already using Vapor; migrations, relationships, multiple DB drivers |
| `GRDB.swift` | Type-safe SQLite access with a nicer Swift API, outside of Vapor |
| Core Data / `SwiftData` | Apple-platform apps needing an object graph + persistence |

The concepts above — prepared statements, parameter binding, row iteration
— are what those libraries do under the hood; understanding this layer
makes their higher-level APIs (and their error messages) far less opaque.

## Swift-specific traps

- **`sqlite3_column_text` returns `nil` for a `NULL` column** — force-
  unwrapping it (as shown above, for brevity) crashes on `NULL` data; real
  code should check for `nil` before calling `String(cString:)`.
- **Forgetting `sqlite3_finalize`** leaks the prepared statement's
  resources and, on some platforms, can leave the database locked — always
  pair `sqlite3_prepare_v2` with a `defer { sqlite3_finalize(statement) }`
  immediately.
- **Bind and column indices are easy to confuse**: bind indices for `?`
  placeholders are 1-based, but column indices when reading a result row
  are 0-based — a frequent off-by-one source.
- **String interpolation into SQL text is never safe**, even for values you
  "control" — always use `?` placeholders and `sqlite3_bind_*`, no
  exceptions.

## Cheat sheet

| Task | API |
|------|-----|
| Open a database | `sqlite3_open(path, &db)` |
| Run DDL/one-off SQL | `sqlite3_exec(db, sql, nil, nil, &errMsg)` |
| Prepare a parameterized statement | `sqlite3_prepare_v2(db, sql, -1, &stmt, nil)` |
| Bind a value | `sqlite3_bind_text(stmt, index, value, -1, SQLITE_TRANSIENT)` |
| Execute an insert/update | `sqlite3_step(stmt) == SQLITE_DONE` |
| Read a query row-by-row | `while sqlite3_step(stmt) == SQLITE_ROW { ... }` |
| Release a statement | `sqlite3_finalize(stmt)` |

## How It Actually Works

**`sqlite3_prepare_v2` compiles SQL text into SQLite's own bytecode
program**, run by a virtual machine internal to the SQLite library — the
`OpaquePointer` you get back (`statement`) refers to that compiled program
plus its execution cursor, not the raw SQL string. Every `sqlite3_step`
call resumes that VM until it either produces a row (`SQLITE_ROW`) or
finishes (`SQLITE_DONE`). This is why preparing once and stepping/binding
repeatedly (e.g. for a batch of inserts, reusing the same prepared
`statement` with `sqlite3_reset` between calls) is dramatically faster than
re-preparing each time: parsing and query planning happen once, at
`prepare_v2`, and every subsequent execution reuses the compiled bytecode.

**Bind parameters are substituted into the bytecode program, never into the
SQL text** — this is the actual mechanism behind "SQL injection is
impossible with `?` placeholders." A string bound via `sqlite3_bind_text`
is stored as opaque data in a VM register and compared/inserted as a value;
it is never re-parsed as SQL syntax, so there is no way for a bound value
containing `; DROP TABLE` to be interpreted as a second statement — it can
only ever be literal text of the query the injected value would supply.
Interpolating into the SQL string before `prepare_v2`, by contrast, hands
attacker-controlled text to SQLite's parser as syntax.

**`OpaquePointer` and manual memory ownership**: Swift's `SQLite3` module
is a direct C interop layer, so none of ARC's automatic retain/release
applies to `db` or `statement` — they're opaque C pointers to SQLite's own
heap allocations, and SQLite (not Swift) owns their lifetime. `defer {
sqlite3_finalize(statement) }` and `defer { sqlite3_close(db) }` exist
because Swift's runtime has no way to know when these resources are no
longer needed — unlike a Swift class instance, there's no reference count
tracking who still points at the prepared statement.

**`SQLITE_TRANSIENT` vs. the alternative `SQLITE_STATIC`** controls whether
SQLite copies the bound string immediately or trusts it to remain valid for
the statement's lifetime; passing a Swift `String`'s temporary C
representation with `SQLITE_STATIC` would be a use-after-free the moment
Swift deallocates that temporary buffer, which is why the module defaults
to the safer (if slightly costlier) transient copy.

## Exercise

Add an `age INTEGER` column to the `employees` table, insert a few rows
with ages, and write a `func averageAge(forRole role: String) -> Double?`
that uses a prepared `SELECT AVG(age) FROM employees WHERE role = ?`
statement (bound with `sqlite3_bind_text`) to compute it. Handle the case
where no employees match the given role by returning `nil`.
