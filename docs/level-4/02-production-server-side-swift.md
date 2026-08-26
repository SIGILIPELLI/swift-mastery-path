# 02 · Production Server-Side Swift

Getting a server-side Swift app *running* (Level 3, Module 02) is different
from getting it *production-ready*: structured logging, environment-driven
configuration, health checks, and graceful shutdown. This module builds
those concerns with `swift-log` — the same structured logging API Vapor
itself uses internally — since it has no C/TLS dependency and compiles
cleanly in any environment.

## Structured logging with `swift-log`

`swift-log` is the community-standard logging API for server-side Swift.
Instead of `print`, you get log levels and structured metadata that a real
log aggregator (Datadog, CloudWatch, etc.) can parse:

```swift
// Package.swift dependency:
// .package(url: "https://github.com/apple/swift-log.git", from: "1.6.0")

import Logging

LoggingSystem.bootstrap(StreamLogHandler.standardOutput)
let logger = Logger(label: "com.example.prodserver")

logger.info("Starting server", metadata: ["port": "8080", "env": "production"])
```

`LoggingSystem.bootstrap` is called exactly once, near the start of
`main`, to choose where logs go — `StreamLogHandler.standardOutput` prints
to stdout, but the same `Logger` API works unchanged if you swap in a
handler that ships logs to a remote aggregator instead.

## Configuration from environment variables

Production services take their configuration (port, database URL, feature
flags) from the environment, not hardcoded values — this is what makes the
same build deployable to staging and production without a recompile:

```swift
import Foundation

struct Config {
    let port: Int
    let environment: String

    static func fromEnvironment() -> Config {
        let port = Int(ProcessInfo.processInfo.environment["PORT"] ?? "") ?? 8080
        let env = ProcessInfo.processInfo.environment["APP_ENV"] ?? "development"
        return Config(port: port, environment: env)
    }
}

let config = Config.fromEnvironment()
logger.info("Starting server", metadata: ["port": "\(config.port)", "env": "\(config.environment)"])
```

Every environment-derived value has a sensible default (`8080`,
`"development"`) so the binary still runs locally without any environment
variables set — production deployments override them via the process
environment (a `docker run -e`, a Kubernetes manifest, a `.env` loaded by
the platform).

## Health checks

A `/health` endpoint (wired to real routing the way Level 3 Module 02
covers) needs a status your infrastructure's load balancer or orchestrator
can act on — `healthy` traffic gets routed to, `unhealthy` gets pulled out
of rotation:

```swift
enum HealthStatus: String { case healthy, degraded, unhealthy }

struct HealthCheck {
    var databaseReachable = true
    var diskSpaceOK = true

    var status: HealthStatus {
        if !databaseReachable { return .unhealthy }
        if !diskSpaceOK { return .degraded }
        return .healthy
    }
}

var health = HealthCheck()
logger.info("Health check", metadata: ["status": "\(health.status.rawValue)"])

health.databaseReachable = false
logger.warning("Database unreachable", metadata: ["status": "\(health.status.rawValue)"])
```

Distinguishing `degraded` from `unhealthy` matters in practice: a
degraded instance (low disk, one of several caches unreachable) might stay
in rotation while paging someone, whereas unhealthy should be pulled
immediately.

## Graceful shutdown

Production deployments send `SIGTERM` (or, interactively, `SIGINT` via
Ctrl-C) before killing a process — catching it lets you finish in-flight
requests and close connections cleanly instead of dropping them:

```swift
let shutdownSource = DispatchSource.makeSignalSource(signal: SIGINT, queue: .main)
signal(SIGINT, SIG_IGN)   // tell the OS's default handler to step aside
shutdownSource.setEventHandler {
    logger.info("Received SIGINT, shutting down gracefully")
    exit(0)
}
shutdownSource.resume()
```

`signal(SIGINT, SIG_IGN)` disables the default "terminate immediately"
behavior so `DispatchSource`'s handler gets a chance to run its own cleanup
logic (flush logs, close database connections, finish in-flight requests)
before the process actually exits.

## Full run

Running the whole program (`PORT=9090 APP_ENV=production swift run`):

```
2026-08-26T22:24:55+0530 info com.example.prodserver: env=production port=9090 [prodserver] Starting server
2026-08-26T22:24:55+0530 info com.example.prodserver: status=healthy [prodserver] Health check
2026-08-26T22:24:55+0530 warning com.example.prodserver: status=unhealthy [prodserver] Database unreachable
2026-08-26T22:24:55+0530 info com.example.prodserver: [prodserver] Server ready, simulating uptime
2026-08-26T22:24:55+0530 info com.example.prodserver: request_id=1 [prodserver] Handling request
2026-08-26T22:24:55+0530 info com.example.prodserver: request_id=2 [prodserver] Handling request
2026-08-26T22:24:55+0530 info com.example.prodserver: request_id=3 [prodserver] Handling request
2026-08-26T22:24:55+0530 info com.example.prodserver: [prodserver] Demo complete, exiting without a real interrupt
```

Each log line carries the level (`info`/`warning`), the logger's label, and
structured `key=value` metadata — exactly the shape a log aggregator's
query language (e.g. filtering on `status=unhealthy`) expects.

## Swift-specific traps

- **`LoggingSystem.bootstrap` can only be called once per process** —
  calling it a second time (say, once in a library and again in the app)
  triggers a runtime crash by design, so libraries should never call it
  themselves; only the executable's `main` should.
- **Metadata values in `swift-log` are `Logger.MetadataValue`, not raw
  strings** — `"\(config.port)"` works because `String` conforms via
  `.stringConvertible`, but passing a raw `Int` directly is a type error;
  string-interpolating simple values, as above, sidesteps this.
- **`ProcessInfo.processInfo.environment` is a snapshot taken once** — it
  doesn't reflect environment variables changed after the process starts,
  which is expected (and desired) for configuration, but surprises people
  expecting live updates.
- **Ignoring `SIGTERM` in favor of only `SIGINT`** works for local `Ctrl-C`
  testing but misses the signal most container orchestrators (Docker,
  Kubernetes) actually send on shutdown — production code should handle
  `SIGTERM` the same way.

## Cheat sheet

| Concern | Tool |
|---------|------|
| Structured logs | `swift-log`'s `Logger` + `LoggingSystem.bootstrap` |
| Environment-driven config | `ProcessInfo.processInfo.environment[...]` with defaults |
| Liveness/readiness signal | A `HealthCheck` type exposing a small enum status |
| Clean shutdown | `DispatchSource.makeSignalSource` for `SIGTERM`/`SIGINT` |

## Exercise

Extend the `HealthCheck` type with a `queueDepth: Int` field and treat
`queueDepth > 100` as `degraded` and `queueDepth > 500` as `unhealthy`
(keeping the existing `databaseReachable` check as an immediate
`unhealthy` regardless of queue depth). Add a `SIGTERM` handler alongside
the existing `SIGINT` one that logs a distinct message before calling
`exit(0)`, and verify both signals produce a graceful log line rather than
an abrupt process kill (you can send `SIGTERM` to your running process with
`kill -TERM <pid>` from another terminal to confirm).
