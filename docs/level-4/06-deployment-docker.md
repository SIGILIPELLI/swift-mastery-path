# 06 · Deployment (Docker)

Docker packages a Swift executable together with its exact runtime
dependencies into a portable image — the same image runs identically on a
laptop, a CI runner, or a cloud VM. This module builds a multi-stage
Dockerfile for a Swift server, then covers image size and runtime
considerations.

!!! note "Environment note for this module"
    This machine doesn't have Docker installed, so nothing below was
    actually built into an image or run in a container here. The
    Dockerfile and commands are hand-reviewed against Swift's official
    Docker images and current `docker build`/`run` syntax — verify them on
    a machine with Docker installed before relying on them.

## Why multi-stage builds

A naive Dockerfile that installs the full Swift toolchain *and* ships it in
the final image produces something enormous (multiple gigabytes) — most of
that is the compiler and build tools, not anything the running server
needs. A multi-stage build compiles in one image and copies only the
resulting binary into a second, minimal one:

```dockerfile
# Stage 1: build
FROM swift:6.0-jammy AS build
WORKDIR /build

# Resolve dependencies first (cached separately from source changes)
COPY Package.swift Package.resolved ./
RUN swift package resolve

# Now copy source and build in release mode
COPY Sources ./Sources
RUN swift build -c release --static-swift-stdlib

# Stage 2: runtime
FROM swift:6.0-jammy-slim
WORKDIR /app

# Copy only the compiled binary from the build stage
COPY --from=build /build/.build/release/restapi /app/restapi

EXPOSE 8123
ENTRYPOINT ["/app/restapi"]
```

`COPY Package.swift Package.resolved ./` followed by `swift package
resolve` *before* copying source code is a deliberate ordering: Docker
caches each layer, so as long as the package manifest hasn't changed,
rebuilding after a source-only edit reuses the cached dependency-resolution
layer instead of re-downloading everything.

## `--static-swift-stdlib`

Statically linking the Swift standard library into the binary means the
runtime image doesn't need a matching Swift installation at all — you could
even copy the binary into a bare `ubuntu` or `distroless` base image rather
than `swift:...-slim`, shrinking the final image further, as long as any
other C library dependencies (like `libsqlite3` for the Module 03/Level 3
database code) are still present in that base image.

## Building and running the image

```bash
docker build -t restapi:latest .
docker run -p 8123:8123 -e APP_ENV=production restapi:latest
```

Expected output (on a machine with Docker):

```
$ docker build -t restapi:latest .
[+] Building 42.1s (12/12) FINISHED
...
=> exporting to image
=> => naming to docker.io/library/restapi:latest

$ docker run -p 8123:8123 -e APP_ENV=production restapi:latest
Server listening on http://127.0.0.1:8123
```

`-p 8123:8123` maps the container's port to the same port on the host;
`-e APP_ENV=production` demonstrates the environment-driven configuration
pattern from Module 02 — the same image behaves differently depending on
what's injected at `docker run` time, with no rebuild needed.

## `.dockerignore`

Excluding build artifacts and version control metadata from the build
context speeds up every `docker build` and avoids accidentally baking
secrets (like a local `.env`) into an image layer:

```
.git
.build
.swiftpm
*.xcodeproj
.env
```

## Health checks in the image

Docker (and orchestrators built on it, like Kubernetes) can poll a
container's health directly, tying back into Module 02's `HealthCheck`
type if the server exposes it over HTTP:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8123/employees || exit 1
```

A container reported unhealthy this way can be automatically restarted or
pulled out of a load balancer's rotation without a human noticing first.

## Swift-specific traps

- **The build-stage and runtime-stage Swift versions should match
  exactly** (`swift:6.0-jammy` build / `swift:6.0-jammy-slim` runtime
  above) — a binary built against one Swift ABI version can fail to run,
  or worse, misbehave subtly, against a mismatched runtime.
- **`--static-swift-stdlib` doesn't statically link *every* dependency** —
  C libraries your package depends on (SQLite, OpenSSL/BoringSSL if using
  Vapor) still need to be present in the runtime base image unless those
  are statically linked too.
- **Forgetting `EXPOSE`** doesn't actually block network access (Docker's
  `EXPOSE` is documentation/metadata, not enforcement) but omitting it
  makes the image's intended ports unclear to anyone deploying it later —
  always declare it even though `-p` at `docker run` time is what actually
  does the port mapping.
- **A `jammy` (non-slim) build image is much larger than needed for the
  final runtime stage** — this is expected and fine, since multi-stage
  builds discard the build stage's layers from the final image; don't
  "optimize" by trying to build directly in the slim image, which lacks
  the full toolchain needed to compile.

## Cheat sheet

| Task | Command/directive |
|------|--------------------|
| Compile for a small runtime footprint | `swift build -c release --static-swift-stdlib` |
| Keep only the compiled binary | Multi-stage `FROM ... AS build` + `COPY --from=build` |
| Build an image | `docker build -t name:tag .` |
| Run with env-driven config | `docker run -p HOST:CONTAINER -e KEY=VALUE image:tag` |
| Exclude files from the build context | `.dockerignore` |
| Let the orchestrator poll liveness | `HEALTHCHECK` directive |

## Stretch goals

- Extend the Dockerfile above to build the REST API + Database project from
  Level 3's capstone, making sure the runtime image includes `libsqlite3`
  (present by default in the `swift:...-slim` images) since that project
  isn't statically linkable the same way a dependency-free binary is.
- Add a `docker-compose.yml` that runs the API container alongside a
  separate, persistent volume for the SQLite file, so data survives a
  container restart.
- Investigate `distroless` base images for the runtime stage (replacing
  `swift:6.0-jammy-slim`) to shrink the final image further, and note what
  breaks (missing shell, missing `curl` for the `HEALTHCHECK` above) that
  you'd need to work around.
