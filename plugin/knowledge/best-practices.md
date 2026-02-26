# Dockerfile Best Practices

Universal Dockerfile guidance applicable to any language or framework. This document is consumed by the `audit`, `create`, and `fix` skills as a reference for evaluating and generating Dockerfiles.

---

## Layer Hygiene

### Why Fewer Layers Matter

Each `RUN`, `COPY`, and `ADD` instruction creates a new layer in the union filesystem. Layers add:
- Filesystem overhead (metadata per layer)
- Increased image size when files are created then deleted in separate layers (the deleted file still exists in the earlier layer)
- Slower pulls and pushes (more layers to transfer and verify)

### Combine RUN Commands with `&&`

**Bad** -- separate RUN instructions:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean
```

**Good** -- single RUN with `&&`:

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Clean Up in the Same Layer

Cleanup in a separate layer does NOT reduce image size. The files persist in the earlier layer.

**Bad** -- cleanup in separate layer:

```dockerfile
RUN dnf install -y httpd
RUN dnf clean all
```

**Good** -- cleanup in same layer:

```dockerfile
RUN dnf install -y httpd && \
    dnf clean all && \
    rm -rf /var/cache/dnf
```

### Common Cleanup Commands by Package Manager

| Package Manager | Cleanup Commands                                        |
|----------------|---------------------------------------------------------|
| apt (Debian)   | `apt-get clean && rm -rf /var/lib/apt/lists/*`          |
| dnf/yum (RHEL) | `dnf clean all && rm -rf /var/cache/dnf`               |
| apk (Alpine)   | `rm -rf /var/cache/apk/*` (or use `--no-cache` flag)   |
| pip (Python)   | `pip install --no-cache-dir`                            |
| npm (Node)     | `npm cache clean --force`                               |

### Heredoc Syntax for Multi-Line RUN

BuildKit supports heredoc syntax for cleaner multi-line commands:

```dockerfile
# syntax=docker/dockerfile:1
RUN <<EOF
set -e
apt-get update
apt-get install -y --no-install-recommends \
    curl \
    ca-certificates
apt-get clean
rm -rf /var/lib/apt/lists/*
EOF
```

Heredocs avoid long `&&` chains and make scripts more readable. Requires BuildKit (`# syntax=docker/dockerfile:1` at top of Dockerfile or `DOCKER_BUILDKIT=1`).

---

## Build Cache Optimization

### How Docker Layer Caching Works

Docker caches each layer by its instruction and inputs. If neither the instruction text nor the input files have changed, Docker reuses the cached layer. A cache miss on any layer invalidates all subsequent layers.

### Ordering: Least-Changing Layers First

Place instructions that change infrequently before those that change often:

1. Base image (`FROM`)
2. System package installation
3. Dependency manifest copy and install
4. Application source code copy
5. Build command

**Bad** -- source code copied before dependencies:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci
```

Any source file change invalidates the `npm ci` cache.

**Good** -- dependency files first, then source:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

Now `npm ci` only re-runs when `package.json` or `package-lock.json` changes.

### Dependency-First Patterns by Language

| Language | Copy First                              | Install Command                    |
|----------|----------------------------------------|------------------------------------|
| Node     | `package.json`, `package-lock.json`    | `npm ci`                           |
| Python   | `requirements.txt` or `pyproject.toml` | `pip install --no-cache-dir -r`    |
| Rust     | `Cargo.toml`, `Cargo.lock`            | `cargo build --release` (see note) |
| Go       | `go.mod`, `go.sum`                     | `go mod download`                  |
| Java     | `pom.xml` or `build.gradle`           | `mvn dependency:go-offline`        |
| Ruby     | `Gemfile`, `Gemfile.lock`             | `bundle install`                   |

> **Rust note:** Rust does not separate dependency download from compilation. Use the `cargo-chef` pattern or a dummy `src/main.rs` to pre-build dependencies. See the multi-stage knowledge base for details.

### Using `.dockerignore` to Prevent Cache Busting

Files irrelevant to the build (`.git`, IDE configs, test artifacts) can trigger unnecessary cache invalidation when using `COPY . .`. A `.dockerignore` excludes them from the build context entirely.

### BuildKit Cache Mounts

Cache mounts persist package manager caches across builds without including them in the final image:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt/lists \
    apt-get update && \
    apt-get install -y --no-install-recommends curl

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release
```

Cache mounts are not included in the image layer. They speed up rebuilds significantly for package managers with large caches.

---

## Secret Management

### Never Put Secrets in ENV, ARG, or COPY

**Critical rule.** Secrets baked into the image are recoverable by anyone with access to the image.

**Bad** -- secrets in ENV:

```dockerfile
ENV DATABASE_URL=postgres://user:password@host/db
ENV API_KEY=sk-live-abc123
```

**Bad** -- secrets in ARG (still visible in image history):

```dockerfile
ARG DB_PASSWORD
RUN echo "password=$DB_PASSWORD" > /etc/app/config
```

**Bad** -- copying secret files:

```dockerfile
COPY .env /app/.env
COPY credentials.json /app/credentials.json
```

All three approaches embed secrets into image layers that can be extracted with `docker history` or `docker save`.

### BuildKit Secrets

BuildKit `--mount=type=secret` makes secrets available at build time without persisting them in any layer:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci

RUN --mount=type=secret,id=pip_conf,target=/etc/pip.conf \
    pip install -r requirements.txt
```

Build invocation:

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
docker build --secret id=pip_conf,src=./pip.conf .
```

The secret is mounted into the RUN instruction's filesystem but never written to a layer.

### Multi-Stage Isolation

Even without BuildKit secrets, multi-stage builds provide isolation. Secrets used in the builder stage do not carry into the runtime stage as long as you do not `COPY` them across:

```dockerfile
FROM node:20-slim AS builder
COPY . .
# Secret used here during build
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
RUN npm run build

FROM node:20-slim AS runtime
# Only the build output is copied -- no secrets
COPY --from=builder /app/dist /app/dist
CMD ["node", "/app/dist/index.js"]
```

### Runtime Secrets

For secrets needed at runtime (database passwords, API keys):
- Use Docker Swarm secrets (`/run/secrets/`) or Kubernetes secrets (mounted volumes)
- Use environment variables injected at `docker run` time (not baked into the image)
- Never embed runtime secrets during build

---

## .dockerignore Patterns

### Why .dockerignore Matters

The build context is the set of files sent to the Docker daemon before the build starts. Without `.dockerignore`:
- `.git` directory adds significant size (the entire repo history)
- `node_modules` or `target/` from local development bloat the context
- `.env` files risk leaking secrets into the build context
- Large test fixtures, documentation, and media waste transfer time

### Essential Excludes

Every project should exclude at minimum:

```
# Version control
.git
.gitignore

# Docker
Dockerfile
Dockerfile*
docker-compose*.yml
.dockerignore

# Environment and secrets
.env
.env.*
*.pem
*.key

# IDE and editor
.vscode
.idea
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db
```

### Language-Specific Patterns

**Node.js:**

```
node_modules
npm-debug.log*
coverage
.nyc_output
dist
```

**Python:**

```
__pycache__
*.pyc
*.pyo
.venv
venv
.pytest_cache
.mypy_cache
*.egg-info
```

**Rust:**

```
target
*.rs.bk
```

**Go:**

```
vendor (if using modules)
*.test
```

**Java:**

```
target
build
*.class
*.jar
.gradle
```

### Keep Patterns (Exceptions)

Use `!` to re-include files that a broad pattern would exclude:

```
*
!src/
!package.json
!package-lock.json
!tsconfig.json
```

This "deny-all, allow-specific" approach is the most secure -- only explicitly listed files enter the build context.

---

## HEALTHCHECK Design

### Why HEALTHCHECK Matters

Without a HEALTHCHECK, Docker (and orchestrators like Swarm) cannot distinguish between a running container and a healthy one. A container can be "running" while the application inside has crashed, deadlocked, or become unresponsive.

HEALTHCHECK enables:
- Automatic restart of unhealthy containers (Docker Swarm, ECS)
- Rolling update safety (new container must be healthy before old one is removed)
- Load balancer integration (unhealthy containers removed from rotation)

> **Note:** Kubernetes uses its own liveness/readiness probes and ignores Dockerfile HEALTHCHECK. However, HEALTHCHECK is still valuable for non-Kubernetes deployments and local development.

### Good HEALTHCHECK Patterns

**HTTP service with curl:**

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["curl", "-f", "http://localhost:8080/health"]
```

**HTTP service with wget (Alpine, no curl):**

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/health"]
```

**HTTP service without curl/wget (distroless/minimal):**

Use a statically compiled health check binary:

```dockerfile
COPY --from=builder /app/healthcheck /usr/local/bin/healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["healthcheck"]
```

### Tuning Parameters

| Parameter        | Default | Guidance                                                     |
|-----------------|---------|--------------------------------------------------------------|
| `--interval`    | 30s     | How often to check. 10-30s for most services.                |
| `--timeout`     | 30s     | Max time for a single check. 3-5s for HTTP, 10s for DB.     |
| `--start-period`| 0s      | Grace period for startup. Set to expected startup time.       |
| `--retries`     | 3       | Failures before "unhealthy". 3 is a good default.            |

- Set `--start-period` generously for JVM apps, Rust binaries with migrations, or anything with slow startup
- Keep `--timeout` short -- a health check that takes 30 seconds defeats the purpose
- Do not set `--interval` below 5s in production (unnecessary load)

### HEALTHCHECK for Non-HTTP Services

**TCP port check:**

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD ["sh", "-c", "nc -z localhost 5432 || exit 1"]
```

**Process check:**

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD ["sh", "-c", "pgrep -x myprocess || exit 1"]
```

**Custom script:**

```dockerfile
COPY healthcheck.sh /usr/local/bin/healthcheck.sh
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD ["healthcheck.sh"]
```

---

## ENTRYPOINT vs CMD

### When to Use Each

| Directive      | Purpose                                                      | Typical Use                        |
|---------------|--------------------------------------------------------------|------------------------------------|
| `ENTRYPOINT`  | Defines the executable. Not easily overridden by `docker run` args. | The main process the container runs. |
| `CMD`         | Provides default arguments. Overridden by `docker run` args. | Default flags or subcommands.      |

### Combined ENTRYPOINT + CMD Pattern

The most robust pattern uses ENTRYPOINT for the binary and CMD for default arguments:

```dockerfile
ENTRYPOINT ["python", "-m", "myapp"]
CMD ["--port", "8080"]
```

Running the container:

```bash
docker run myapp                     # runs: python -m myapp --port 8080
docker run myapp --port 9090         # runs: python -m myapp --port 9090
docker run --entrypoint sh myapp     # overrides entrypoint to get a shell
```

### Exec Form vs Shell Form

**Exec form** (preferred):

```dockerfile
ENTRYPOINT ["node", "server.js"]
CMD ["--port", "8080"]
```

**Shell form** (avoid in most cases):

```dockerfile
ENTRYPOINT node server.js
CMD --port 8080
```

Key differences:

| Aspect              | Exec Form                    | Shell Form                         |
|--------------------|-----------------------------|------------------------------------|
| PID 1              | The specified process       | `/bin/sh -c` wraps the process     |
| Signal handling    | SIGTERM reaches the process | SIGTERM goes to shell, not process |
| Variable expansion | No (`$VAR` not expanded)    | Yes (`$VAR` is expanded)           |
| Graceful shutdown  | Works correctly             | Broken -- container gets SIGKILL after timeout |

### Why Exec Form Is Preferred

1. **PID 1 and signal handling:** When using shell form, `/bin/sh -c` becomes PID 1 and does not forward signals. The application never receives SIGTERM, so Docker kills it with SIGKILL after the stop timeout (default 10s). This means no graceful shutdown, no connection draining, no cleanup.

2. **No zombie processes:** In exec form, the application is PID 1 and can properly reap child processes (or use `tini` / `dumb-init` as an init process).

3. **Predictable behavior:** No shell interpretation surprises with special characters or variable expansion.

**If you need variable expansion with exec form**, use a shell explicitly:

```dockerfile
ENTRYPOINT ["sh", "-c", "exec myapp --config=$CONFIG_PATH"]
```

The `exec` replaces the shell process so the application becomes PID 1.

---

## ARG vs ENV Scoping

### Build-Time ARG vs Runtime ENV

| Directive | Available at build time | Available at runtime | Persisted in image |
|-----------|------------------------|---------------------|--------------------|
| `ARG`     | Yes                    | No                  | No (but see cache note) |
| `ENV`     | Yes                    | Yes                 | Yes                |

### ARG Scope Across Stages

An `ARG` defined before `FROM` is only available in `FROM` directives. An `ARG` defined after `FROM` is scoped to that stage:

```dockerfile
# Global ARG -- only usable in FROM
ARG RUST_VERSION=1.77

FROM rust:${RUST_VERSION}-slim AS builder
# Must re-declare to use within the stage
ARG RUST_VERSION
RUN echo "Building with Rust ${RUST_VERSION}"

FROM debian:bookworm-slim AS runtime
# RUST_VERSION is NOT available here unless re-declared
```

### Using ARG for Version Pinning

Pin versions at the top of the Dockerfile for easy updates:

```dockerfile
ARG NODE_VERSION=20.11
ARG ALPINE_VERSION=3.19

FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS builder
# ...

FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION} AS runtime
# ...
```

Override at build time:

```bash
docker build --build-arg NODE_VERSION=22.1 .
```

### Common Pitfalls

**ARG values are cache keys.** Changing a build arg invalidates the cache for all subsequent layers that depend on it:

```dockerfile
ARG BUILD_DATE
# Every layer after this is cache-busted on every build
RUN echo "Built on ${BUILD_DATE}" && npm ci
```

**Fix:** Place volatile ARGs as late as possible:

```dockerfile
RUN npm ci
ARG BUILD_DATE
LABEL build-date=${BUILD_DATE}
```

**ENV persists in the final image.** Do not use ENV for build-only values:

```dockerfile
# Bad -- DEBIAN_FRONTEND persists in the runtime image
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y curl

# Good -- scoped to the single RUN
RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y curl

# Also good -- ARG does not persist
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y curl
```

---

## Reproducible Builds

### Pinning Base Image Digests vs Tags

Tags are mutable. `node:20-slim` today may point to a different image tomorrow.

**Acceptable** -- pinned minor version tag:

```dockerfile
FROM node:20.11-slim
```

**Best** -- pinned by digest (immutable):

```dockerfile
FROM node:20.11-slim@sha256:abc123...
```

Use `docker pull <image>` and note the digest, or look it up on Docker Hub. Digest pinning guarantees byte-for-byte reproducibility.

**Trade-off:** Digest pinning requires manual updates to get security patches. Use Dependabot, Renovate, or similar tooling to automate digest updates.

### Pinning Package Versions

**Bad** -- unpinned packages:

```dockerfile
RUN apt-get install -y curl python3
RUN pip install flask requests
```

**Good** -- pinned packages:

```dockerfile
RUN apt-get install -y \
    curl=7.88.1-10+deb12u5 \
    python3=3.11.2-6
RUN pip install --no-cache-dir \
    flask==3.0.2 \
    requests==2.31.0
```

For system packages, at minimum pin to the major version. For application dependencies, use a lockfile (see below).

### Checksum Verification for Downloaded Binaries

Always verify checksums when downloading binaries in a Dockerfile:

```dockerfile
ARG TINI_VERSION=v0.19.0
ARG TINI_SHA256=93dcc18adc78c65a028a84799ecf8ad40c936fdfc5f2a57b1acda5a8117fa82c

RUN curl -fsSL "https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini" \
        -o /usr/local/bin/tini && \
    echo "${TINI_SHA256}  /usr/local/bin/tini" | sha256sum -c - && \
    chmod +x /usr/local/bin/tini
```

For GPG-signed releases, verify signatures:

```dockerfile
RUN curl -fsSL https://example.com/binary -o /tmp/binary && \
    curl -fsSL https://example.com/binary.asc -o /tmp/binary.asc && \
    gpg --verify /tmp/binary.asc /tmp/binary
```

### Lockfile Flags for Package Managers

Use strict install modes that respect lockfiles and fail on mismatch:

| Language | Strict Install Command           | What It Does                                     |
|----------|----------------------------------|--------------------------------------------------|
| Node     | `npm ci`                         | Installs from `package-lock.json` exactly         |
| Rust     | `cargo build --locked`           | Fails if `Cargo.lock` is out of sync              |
| Python   | `pip install -r requirements.txt` (with hashes) | Lockfile with `--require-hashes`    |
| Go       | `go mod download`                | Downloads exact versions from `go.sum`            |
| Ruby     | `bundle install --frozen`        | Fails if `Gemfile.lock` is out of date            |
| PHP      | `composer install --no-dev`      | Installs from `composer.lock`                     |

### Avoiding `latest` Everywhere

The word `latest` should never appear in a production Dockerfile:

**Bad:**

```dockerfile
FROM python:latest
RUN pip install flask
```

**Good:**

```dockerfile
FROM python:3.12.2-slim-bookworm
RUN pip install --no-cache-dir flask==3.0.2
```

`latest` introduces three problems:
1. **Non-reproducible builds** -- the same Dockerfile produces different images on different days
2. **Silent breaking changes** -- a major version bump can break your application
3. **Cache unpredictability** -- Docker may or may not pull a new version depending on local cache state
