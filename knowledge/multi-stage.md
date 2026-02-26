# Multi-Stage Build Patterns

Multi-stage builds are the single most impactful technique for reducing image size and improving security. Every production Dockerfile should use at least two stages: one to build, one to run.

**Core principle:** Build tools, compilers, headers, and dev dependencies never ship in the runtime image. If it is not needed at runtime, it stays in the build stage.

---

## Pattern 1: Basic Builder + Runtime

The simplest multi-stage pattern. Build in one stage, copy only the binary to a minimal runtime.

### Go (static binary)

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

**Size impact:** ~800MB (golang:1.22) down to ~2MB (distroless/static + binary). Saves **~798MB**.

### Rust (static with musl)

```dockerfile
FROM rust:1.77-slim AS builder
RUN apt-get update && apt-get install -y musl-tools && rm -rf /var/lib/apt/lists/*
RUN rustup target add x86_64-unknown-linux-musl
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs && \
    cargo build --release --target x86_64-unknown-linux-musl && \
    rm -rf src
COPY src ./src
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
USER 65534:65534
ENTRYPOINT ["/myapp"]
```

**Size impact:** ~1.5GB (rust:1.77-slim + deps) down to ~5-15MB (scratch + binary). Saves **~1.4GB**.

### Java (JRE runtime)

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Size impact:** ~800MB (JDK + build tools) down to ~150MB (JRE-alpine). Saves **~650MB**.

---

## Pattern 2: Dependency Caching Stage

Separate dependency installation from source compilation. Dependencies change less frequently than source code, so this layer gets cached across builds.

**Rule of thumb:** COPY only the dependency manifest first, install, then COPY source.

### Go

```dockerfile
FROM golang:1.22 AS deps
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

FROM deps AS builder
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/server ./cmd/server

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

### Node.js

```dockerfile
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

FROM node:20-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
CMD ["dist/index.js"]
```

**Cache benefit:** Changing source code does not re-download node_modules. Saves 30-120s on typical builds.

### Python

```dockerfile
FROM python:3.12-slim AS deps
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=deps /install /usr/local
COPY . .
RUN adduser --system --no-create-home appuser
USER appuser
ENTRYPOINT ["python", "app.py"]
```

### Rust (cargo-chef)

```dockerfile
FROM rust:1.77-slim AS chef
RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN adduser --system --no-create-home appuser
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
USER appuser
ENTRYPOINT ["myapp"]
```

**Cache benefit:** `cargo chef cook` downloads and compiles all dependencies from the recipe. Source changes only trigger the final `cargo build`. Saves **5-30 minutes** on large Rust projects.

---

## Pattern 3: Rootfs-Builder Pattern

For maximum security. Build a minimal root filesystem using `--installroot`, then inject it into `ubi-micro`. This produces images smaller and more auditable than even `ubi-minimal`.

```dockerfile
# Stage 1: Build a minimal rootfs with only the packages we need
FROM registry.access.redhat.com/ubi9/ubi AS rootfs-builder
RUN mkdir -p /mnt/rootfs && \
    dnf install \
        --installroot /mnt/rootfs \
        --releasever 9 \
        --setopt install_weak_deps=0 \
        --nodocs \
        -y \
        glibc-minimal-langpack \
        ca-certificates \
        openssl-libs \
        shadow-utils \
    && dnf --installroot /mnt/rootfs clean all \
    && rm -rf /mnt/rootfs/var/cache/* \
              /mnt/rootfs/var/log/dnf* \
              /mnt/rootfs/var/log/yum*

# Stage 2: Copy rootfs into ubi-micro (which has no package manager)
FROM registry.access.redhat.com/ubi9/ubi-micro
COPY --from=rootfs-builder /mnt/rootfs /
```

### Why this beats ubi-minimal

| Aspect | ubi-minimal (~100MB) | rootfs-builder + ubi-micro (~35MB) |
|--------|--------------------|------------------------------------|
| Package manager | microdnf present | None |
| Installed packages | Default minimal set | Only what you explicitly listed |
| Attack surface | ~80 packages | ~10-15 packages |
| STIG compliance | Partial | Full control over every package |

**Size impact:** Saves **~65MB** over ubi-minimal and removes the package manager from the runtime image entirely.

### Rootfs-Builder with STIG Hardening

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi AS rootfs-builder
RUN mkdir -p /mnt/rootfs && \
    dnf install \
        --installroot /mnt/rootfs \
        --releasever 9 \
        --setopt install_weak_deps=0 \
        --nodocs \
        -y \
        glibc-minimal-langpack \
        ca-certificates \
        openssl-libs \
        shadow-utils \
    && dnf --installroot /mnt/rootfs clean all \
    && rm -rf /mnt/rootfs/var/cache/*

# STIG hardening inside the rootfs
RUN echo "* hard core 0" >> /mnt/rootfs/etc/security/limits.d/core.conf && \
    echo "* hard maxlogins 10" >> /mnt/rootfs/etc/security/limits.d/maxlogins.conf && \
    sed -i 's/umask 022/umask 077/' /mnt/rootfs/etc/profile && \
    sed -i 's/umask 002/umask 077/' /mnt/rootfs/etc/bashrc 2>/dev/null || true && \
    sed -i '/nullok/d' /mnt/rootfs/etc/pam.d/* 2>/dev/null || true && \
    echo "" > /mnt/rootfs/etc/machine-id

FROM registry.access.redhat.com/ubi9/ubi-micro
COPY --from=rootfs-builder /mnt/rootfs /
```

---

## Pattern 4: Scanner Integration Stage

Bake security scanners into the build pipeline. Scanners run during build, and optionally their binaries are available in the runtime image for on-demand scanning.

```dockerfile
# Stage 1: Download scanner binaries
FROM registry.access.redhat.com/ubi9/ubi-minimal AS scanners
RUN microdnf install -y tar gzip && \
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /opt/bin v0.50.1 && \
    curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /opt/bin v0.74.0

# Stage 2: Build the application
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

# Stage 3: Build the runtime image
FROM gcr.io/distroless/static:nonroot AS runtime
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# Stage 4: Scan the runtime image (runs during build, fails build on critical vulns)
FROM runtime AS scanned
COPY --from=scanners /opt/bin/trivy /trivy
COPY --from=scanners /opt/bin/grype /grype
USER root
RUN /trivy filesystem --exit-code 1 --severity CRITICAL,HIGH --no-progress / || true
RUN /grype dir:/ --fail-on critical || true
USER nonroot
```

**Build command:** `docker build --target runtime -t myapp:latest .`

The `scanned` stage runs only when explicitly targeted (`--target scanned`). This integrates vulnerability scanning into CI without bloating the production image.

### Scanner-only CI Stage

For CI pipelines that scan without including scanners in any runtime target:

```dockerfile
FROM aquasec/trivy:0.50.1 AS trivy-scan
COPY --from=runtime / /scan-root
RUN trivy rootfs --exit-code 1 --severity CRITICAL /scan-root
```

---

## Pattern 5: Asset Compilation Stage

For applications with separate frontend and backend builds. Each build runs in its own optimized stage.

### Node Frontend + Python Backend

```dockerfile
FROM node:20-slim AS frontend
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

FROM python:3.12-slim AS backend
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
COPY --from=frontend /app/dist /app/static
RUN adduser --system --no-create-home appuser
USER appuser
EXPOSE 8000
ENTRYPOINT ["gunicorn", "app:app", "--bind", "0.0.0.0:8000"]
```

**Size impact:** Node.js toolchain (~400MB) stays out of the Python runtime image. Only the built static assets (~5-20MB) are copied.

### Node Frontend + Go Backend

```dockerfile
FROM node:20-slim AS frontend
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
COPY --from=frontend /app/dist ./static
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/server

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**Size impact:** Node toolchain (~400MB) + Go toolchain (~800MB) down to ~5MB distroless + binary. Saves **~1.2GB**.

---

## Stage Naming Conventions

Use descriptive stage names. Never rely on numeric indices.

### Recommended Names

| Stage purpose | Name | Avoid |
|---------------|------|-------|
| Compile source | `builder` | `build`, `stage1` |
| Install dependencies | `deps` | `dependencies`, `stage0` |
| Plan dependency graph | `planner` | `plan` |
| Build minimal rootfs | `rootfs-builder` | `rootfs`, `base` |
| Download tools/scanners | `scanners` | `tools`, `utils` |
| Compile frontend assets | `frontend` | `ui`, `client` |
| Final runtime image | `runtime` | `final`, `production` |

### Naming Rules

- Use lowercase with hyphens: `rootfs-builder`, not `RootfsBuilder`
- Reference stages by name: `COPY --from=builder`, not `COPY --from=0`
- Keep stages in logical order: dependencies, build, runtime
- Name every stage, even if you think there are only two

```dockerfile
# Good: named stages, clear purpose
FROM golang:1.22 AS builder
FROM gcr.io/distroless/static AS runtime

# Bad: unnamed stages, fragile numeric references
FROM golang:1.22
FROM gcr.io/distroless/static
COPY --from=0 /app/server /server
```

---

## COPY --from References

### From a Named Stage

```dockerfile
COPY --from=builder /app/server /server
COPY --from=frontend /app/dist /static
```

### From an External Image

Pull specific files from published images without adding them as a stage:

```dockerfile
COPY --from=busybox:1.36-musl /bin/sh /bin/sh
COPY --from=alpine:3.19 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=aquasec/trivy:0.50.1 /usr/local/bin/trivy /usr/local/bin/trivy
```

### Copy Discipline

Copy specific files, not entire directories:

```dockerfile
# Good: copy only the binary
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp

# Bad: copy the entire build directory (includes sources, intermediate artifacts)
COPY --from=builder /app /app

# Good: copy only the installed dependencies
COPY --from=deps /install /usr/local

# Bad: copy the entire filesystem
COPY --from=deps / /
```

---

## Runtime Base Image Selection

Choose the runtime base image based on what the application actually needs.

### Decision Matrix

| Base Image | Size | Shell | Package Mgr | libc | Use When |
|------------|------|-------|-------------|------|----------|
| `scratch` | 0 MB | No | No | No | Static binaries (Go with `CGO_ENABLED=0`, Rust with musl) |
| `gcr.io/distroless/static` | ~2 MB | No | No | No | Static binaries + CA certs + timezone data |
| `gcr.io/distroless/base` | ~20 MB | No | No | glibc | Need glibc but nothing else |
| `gcr.io/distroless/cc` | ~25 MB | No | No | glibc + libstdc++ | C++ applications, Rust with glibc |
| `alpine:3.19` | ~7 MB | Yes | apk | musl | Need shell, want minimal, musl-compatible |
| `debian:bookworm-slim` | ~80 MB | Yes | apt | glibc | Need glibc + broad package support |
| `ubi9/ubi-micro` | ~30 MB | No | No | glibc | RHEL compat, FIPS-capable, enterprise support |
| `ubi9/ubi-minimal` | ~100 MB | Yes | microdnf | glibc | Need runtime package installs on RHEL |

### Selection Rules

1. **Can you compile a fully static binary?** Use `scratch` or `distroless/static`.
2. **Need glibc but no shell?** Use `distroless/base` or `distroless/cc`.
3. **Need RHEL/FIPS compliance?** Use `ubi-micro` with rootfs-builder pattern.
4. **Need a shell for debugging?** Use `alpine` (small) or `debian-slim` (compatible).
5. **Need to install packages at runtime?** Use `ubi-minimal` or `debian-slim` (but avoid this pattern when possible).

### Size Comparison: Same Go App on Different Bases

| Base | Final Image Size | Attack Surface |
|------|-----------------|----------------|
| `golang:1.22` (no multi-stage) | ~850 MB | Very high (compiler, tools, headers) |
| `ubuntu:24.04` | ~82 MB | High (full OS) |
| `debian:bookworm-slim` | ~85 MB | Medium (shell, apt, coreutils) |
| `alpine:3.19` | ~12 MB | Low (shell, apk, busybox) |
| `gcr.io/distroless/static` | ~7 MB | Very low (CA certs, tzdata only) |
| `scratch` | ~5 MB | Minimal (binary only) |

Binary size assumed: ~5MB static Go binary.

---

## Anti-Patterns

### Installing build tools in the runtime image

```dockerfile
# Bad: gcc and build dependencies ship in the final image
FROM python:3.12
RUN apt-get update && apt-get install -y gcc libffi-dev
RUN pip install cryptography
CMD ["python", "app.py"]
```

```dockerfile
# Good: build in one stage, run in another
FROM python:3.12 AS builder
RUN apt-get update && apt-get install -y gcc libffi-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
COPY --from=builder /install /usr/local
COPY . /app
CMD ["python", "/app/app.py"]
```

**Size impact:** Removes ~300MB of build tools from the runtime image.

### Copying the entire build context into runtime

```dockerfile
# Bad: source code, tests, docs all in runtime image
COPY --from=builder /app /app

# Good: copy only the compiled output
COPY --from=builder /app/dist /app/dist
COPY --from=builder /app/package.json /app/package.json
```

### Using a single stage for everything

```dockerfile
# Bad: 1.2GB image with Go compiler, git, build cache
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server
CMD ["./server"]
```

Split into builder + runtime. Every single-stage Dockerfile is a candidate for multi-stage conversion.

---

## Quick Reference: Minimal Multi-Stage Templates

### Go

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /server

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

### Rust

```dockerfile
FROM rust:1.77-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs && cargo build --release && rm -rf src
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
ENTRYPOINT ["myapp"]
```

### Python

```dockerfile
FROM python:3.12-slim AS builder
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
COPY --from=builder /install /usr/local
COPY . /app
WORKDIR /app
ENTRYPOINT ["python", "app.py"]
```

### Node.js

```dockerfile
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

FROM node:20-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
CMD ["dist/index.js"]
```

### Java

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
COPY --from=builder /app/target/*.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### UBI9 (STIG-grade, any language)

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi AS rootfs-builder
RUN mkdir -p /mnt/rootfs && \
    dnf install --installroot /mnt/rootfs --releasever 9 \
        --setopt install_weak_deps=0 --nodocs -y \
        glibc-minimal-langpack ca-certificates openssl-libs shadow-utils \
    && dnf --installroot /mnt/rootfs clean all \
    && rm -rf /mnt/rootfs/var/cache/*
# Apply STIG hardening
RUN echo "* hard core 0" >> /mnt/rootfs/etc/security/limits.d/core.conf && \
    sed -i 's/umask 022/umask 077/' /mnt/rootfs/etc/profile && \
    echo "" > /mnt/rootfs/etc/machine-id

FROM registry.access.redhat.com/ubi9/ubi-micro
COPY --from=rootfs-builder /mnt/rootfs /
COPY --from=builder /app/myapp /usr/local/bin/myapp
USER 1001:1001
ENTRYPOINT ["myapp"]
```
