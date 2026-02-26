# Security Tiers

This document defines every Dockerfile security rule used by the secure-dockerfile plugin. Skills read this file to determine what to check during audits, what to enforce during creation, and what to fix during hardening.

## Tier Overview

| Tier | Prefix | Description | Includes |
|------|--------|-------------|----------|
| **Standard** | SD | Baseline every production Dockerfile should meet. Focuses on reproducibility, size, and basic hygiene. | -- |
| **Hardened** | HD | Defense-in-depth container security. Minimizes attack surface and follows principle of least privilege. | All Standard rules |
| **STIG** | ST | Meets DoD STIG-level hardening. Includes OS-level controls, FIPS crypto, and audit traceability. | All Standard + Hardened rules |

## Severity Levels

| Severity | Audit Score Deduction | Meaning |
|----------|-----------------------|---------|
| Critical | -15 | Exploitable in production; must fix before shipping |
| High | -10 | Significant security or size risk; fix soon |
| Medium | -5 | Best practice violation; improves quality |
| Low | -2 | Minor improvement; nice to have |

---

## Standard Tier Rules

### SD001: Pinned Base Image Tags

- **Severity:** Critical
- **Tier:** Standard
- **Description:** Base images must use a specific version tag, not `latest` or no tag. Unpinned tags cause non-reproducible builds and may silently introduce breaking changes or vulnerabilities.

**Bad example:**
```dockerfile
FROM ubuntu:latest
FROM node
FROM python:3
```

**Good example:**
```dockerfile
FROM ubuntu:24.04
FROM node:20.11-slim
FROM python:3.12.1-slim-bookworm
```

**Rationale:** Using `latest` or bare tags means every build may pull a different image. A base image update could introduce new vulnerabilities, break your application, or change the OS entirely. Pinned tags ensure reproducible builds and make vulnerability tracking possible. For maximum reproducibility, consider pinning to a digest (`FROM ubuntu@sha256:...`).

---

### SD002: Multi-Stage Builds

- **Severity:** High
- **Tier:** Standard
- **Description:** Dockerfiles should use multi-stage builds to separate build-time dependencies (compilers, build tools, dev headers) from the runtime image. Single-stage builds ship unnecessary tools that increase image size and attack surface.

**Bad example:**
```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o myapp .
EXPOSE 8080
CMD ["./myapp"]
```

**Good example:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

FROM gcr.io/distroless/static:nonroot
WORKDIR /app
COPY --from=builder /app/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

**Rationale:** A single-stage Go image with the full SDK is ~800MB. The multi-stage version using distroless is ~15MB. Beyond size, build tools (gcc, make, git) in runtime images provide attackers with utilities to compile exploits, download payloads, or pivot within the container.

---

### SD003: Combined RUN Layers

- **Severity:** Medium
- **Tier:** Standard
- **Description:** Package update and install commands must be combined into a single RUN instruction. Separate `RUN apt-get update` and `RUN apt-get install` layers cause stale package index caching and create unnecessary intermediate layers.

**Bad example:**
```dockerfile
RUN apt-get update
RUN apt-get install -y curl wget
RUN apt-get install -y nginx
```

**Good example:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        wget \
        nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Rationale:** When `apt-get update` runs in a separate layer, Docker may cache it even when the package list changes upstream. Subsequent `apt-get install` commands then use stale indexes, leading to missing packages or version mismatches. Combining them ensures the index is always fresh when packages are installed. Each additional layer also adds to the total image size.

---

### SD004: Cache Cleanup in Same Layer

- **Severity:** Medium
- **Tier:** Standard
- **Description:** Package manager caches must be removed in the same RUN layer that installs packages. Cleaning up in a separate layer does not reduce image size because the cache still exists in the earlier layer.

**Bad example:**
```dockerfile
RUN dnf install -y httpd php
RUN dnf clean all
RUN rm -rf /var/cache/dnf
```

**Good example:**
```dockerfile
RUN dnf install -y httpd php && \
    dnf clean all && \
    rm -rf /var/cache/dnf
```

**Rationale:** Docker images are built as a stack of layers. Each layer captures filesystem changes. If you install 200MB of packages in layer N and delete caches in layer N+1, the image still contains the 200MB cache in layer N. Cleanup must happen in the same RUN instruction to benefit from layer squashing. This applies to all package managers: `apt-get clean && rm -rf /var/lib/apt/lists/*`, `dnf clean all && rm -rf /var/cache/dnf`, `apk cache clean`, `pip cache purge`, `npm cache clean --force`.

---

### SD005: .dockerignore Present

- **Severity:** Medium
- **Tier:** Standard
- **Description:** A `.dockerignore` file must exist alongside the Dockerfile to prevent unnecessary or sensitive files from being sent to the build context and potentially copied into the image.

**Bad example:**
```
(no .dockerignore file present)
```

**Good example:**
```dockerignore
.git
.gitignore
.env
.env.*
*.md
LICENSE
docker-compose*.yml
.dockerignore
node_modules
__pycache__
*.pyc
target/
build/
dist/
.idea/
.vscode/
*.swp
*.swo
```

**Rationale:** Without `.dockerignore`, the entire project directory is sent as build context, including `.git` (which can be hundreds of MB), `node_modules`, compiled artifacts, and sensitive files like `.env` containing secrets. This slows builds, bloats images, and risks leaking credentials. A `COPY . .` instruction without `.dockerignore` is especially dangerous because it copies everything into the image layer.

---

### SD006: No Secrets in ENV/ARG/COPY

- **Severity:** Critical
- **Tier:** Standard
- **Description:** Secrets (passwords, API keys, tokens, private keys) must never appear in `ENV`, `ARG`, or be copied via `COPY`/`ADD` instructions. Image layers are permanent and can be extracted by anyone with access to the image.

**Bad example:**
```dockerfile
ENV DB_PASSWORD=supersecret123
ARG AWS_SECRET_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
COPY .env /app/.env
COPY credentials.json /app/credentials.json
RUN echo "machine github.com login token password ghp_xxxx" > ~/.netrc
```

**Good example:**
```dockerfile
# Use BuildKit secrets for build-time needs
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc \
    npm ci --production

# Pass secrets at runtime via environment variables or mounted volumes
# docker run -e DB_PASSWORD -v /run/secrets:/run/secrets myapp

# Or use Docker secrets in Swarm / Kubernetes secrets
ENV DB_HOST=db.internal
ENV DB_PORT=5432
# DB_PASSWORD injected at runtime, never baked into image
```

**Rationale:** Every instruction in a Dockerfile creates a layer in the image. Even if you delete a secret in a later layer, it remains in the history and can be extracted with `docker history` or by inspecting the image filesystem. ARG values are visible in `docker inspect`. Secrets must be injected at runtime (environment variables, mounted files, orchestrator secrets) or via BuildKit `--mount=type=secret` for build-time needs.

---

### SD007: HEALTHCHECK Defined

- **Severity:** Medium
- **Tier:** Standard
- **Description:** Dockerfiles should include a HEALTHCHECK instruction so container orchestrators can detect when the application is unhealthy and take corrective action (restart, reroute traffic).

**Bad example:**
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci --production
EXPOSE 3000
CMD ["node", "server.js"]
```

**Good example:**
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci --production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["node", "-e", "require('http').get('http://localhost:3000/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1) })"]
CMD ["node", "server.js"]
```

**Rationale:** Without a HEALTHCHECK, Docker and orchestrators have no way to know if the application inside the container is actually serving traffic. The container may be running (PID 1 alive) but the app could be deadlocked, out of memory, or returning errors. HEALTHCHECK enables automatic recovery. Note: Kubernetes uses its own liveness/readiness probes, so this is most critical for Docker Compose and Docker Swarm deployments.

---

### SD008: Specific COPY Paths

- **Severity:** Low
- **Tier:** Standard
- **Description:** Prefer copying specific files and directories instead of using `COPY . .`, which copies the entire build context and can inadvertently include sensitive or unnecessary files.

**Bad example:**
```dockerfile
WORKDIR /app
COPY . .
RUN npm ci && npm run build
```

**Good example:**
```dockerfile
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
COPY public/ ./public/
RUN npm run build
```

**Rationale:** `COPY . .` copies everything in the build context into the image, including files that `.dockerignore` might not cover (test fixtures, documentation, CI configs). Specific copies also improve build cache utilization: dependency manifests can be copied first, installed, then source code copied. This way, dependency installation is only re-run when manifests change, not on every source code edit.

---

### SD009: Layer Ordering for Cache

- **Severity:** Low
- **Tier:** Standard
- **Description:** Dockerfile instructions should be ordered from least frequently changing to most frequently changing. Dependencies should be installed before application source code is copied.

**Bad example:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

**Good example:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Rationale:** Docker caches each layer and invalidates the cache from the first changed layer onward. If source code (which changes constantly) is copied before `pip install` (which changes rarely), every code change triggers a full dependency reinstall. Copying dependency manifests first and installing them ensures the dependency layer is cached across most builds, reducing build times from minutes to seconds.

---

### SD010: Minimal Base Image Selection

- **Severity:** Medium
- **Tier:** Standard
- **Description:** Use the smallest appropriate base image for the runtime stage. Full OS images include unnecessary packages, libraries, and tools that increase size and attack surface.

**Bad example:**
```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y python3
COPY app.py .
CMD ["python3", "app.py"]
```

**Good example:**
```dockerfile
FROM python:3.12-slim-bookworm
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python3", "app.py"]
```

**Rationale:** `ubuntu:24.04` is ~78MB and includes systemd, login utilities, and many packages irrelevant to running a Python app. `python:3.12-slim-bookworm` is ~52MB and purpose-built. Alpine variants are even smaller (~5MB base). For compiled languages, `scratch` or `distroless` images (which contain only the runtime and TLS certificates) reduce size to single-digit MB and eliminate nearly all CVEs from the base layer.

---

## Hardened Tier Rules

> The Hardened tier includes all Standard tier rules plus the following.

### HD001: Non-Root USER

- **Severity:** Critical
- **Tier:** Hardened
- **Description:** The final image must not run as root. A `USER` instruction must switch to an unprivileged user before `CMD`/`ENTRYPOINT`. Running as root inside a container gives an attacker full control if they escape the container's namespaces.

**Bad example:**
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci --production
EXPOSE 3000
CMD ["node", "server.js"]
# Runs as root by default
```

**Good example:**
```dockerfile
FROM node:20-slim
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser
WORKDIR /app
COPY --chown=appuser:appuser . .
RUN npm ci --production
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

**Rationale:** Running as root means any vulnerability in the application (RCE, path traversal) gives the attacker uid 0 inside the container. With user namespaces disabled (the default in most Docker installations), root inside the container is root on the host. Even with namespaces, root can modify any file in the container filesystem, install tools, and pivot more easily. Always create a dedicated user with minimal permissions.

---

### HD002: Read-Only Filesystem Friendly

- **Severity:** Medium
- **Tier:** Hardened
- **Description:** The image should be designed to run with a read-only root filesystem (`--read-only`). Any writable paths should be explicitly declared as volumes or tmpfs mounts, not scattered throughout the filesystem.

**Bad example:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
# App writes logs to /app/logs, cache to /app/cache, temp to /tmp
CMD ["python", "app.py"]
```

**Good example:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt && \
    mkdir -p /app/logs /app/cache
VOLUME ["/app/logs", "/app/cache"]
# Run with: docker run --read-only --tmpfs /tmp myapp
CMD ["python", "app.py"]
```

**Rationale:** A read-only root filesystem prevents attackers from modifying application binaries, writing web shells, or persisting malware. It enforces immutable infrastructure principles. Explicit VOLUME declarations document which paths need to be writable, making the container's I/O behavior transparent and auditable. Use `--tmpfs /tmp` for temporary files that do not need persistence.

---

### HD003: No Shell in Final Image

- **Severity:** Medium
- **Tier:** Hardened
- **Description:** The final runtime image should not contain a shell (`/bin/sh`, `/bin/bash`) when possible. Shells enable attackers to run arbitrary commands if they achieve code execution.

**Bad example:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

FROM debian:bookworm-slim
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

**Good example:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/myapp /
CMD ["/myapp"]
```

**Rationale:** Shells are the first tool an attacker uses after gaining code execution. Without a shell, an attacker cannot pipe commands, start reverse shells, or easily navigate the filesystem. Distroless and scratch images contain no shell, no package manager, and no common Unix utilities. This dramatically limits what an attacker can do even after achieving RCE. Use exec-form CMD (JSON array) since shell-form requires `/bin/sh`.

---

### HD004: Minimal Packages

- **Severity:** Medium
- **Tier:** Hardened
- **Description:** Package installation must suppress recommended and suggested packages. On Debian/Ubuntu use `--no-install-recommends`. On RHEL/Fedora use `install_weak_deps=0`. Every additional package increases attack surface.

**Bad example:**
```dockerfile
RUN apt-get update && \
    apt-get install -y \
        python3 \
        python3-pip && \
    rm -rf /var/lib/apt/lists/*
```

**Good example:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip && \
    rm -rf /var/lib/apt/lists/*
```

**Rationale:** Without `--no-install-recommends`, apt installs "Recommends" packages automatically. A simple `python3` install can pull in X11 libraries, documentation packages, and GUI tools, adding 50-100MB and dozens of additional CVE-bearing packages. On RHEL systems, `install_weak_deps=0` in `dnf.conf` achieves the same effect. Every unnecessary package is a potential vulnerability and increases the image size.

---

### HD005: Drop All Capabilities Guidance

- **Severity:** High
- **Tier:** Hardened
- **Description:** The Dockerfile or its comments should document that the container must be run with `--cap-drop=ALL` and only add back the specific capabilities required. Linux capabilities grant fine-grained root-like powers that should be minimized.

**Bad example:**
```dockerfile
FROM nginx:1.25-alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
# No guidance on capabilities
```

**Good example:**
```dockerfile
FROM nginx:1.25-alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 8080
# Run with: docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
# Or for non-privileged port (8080), no capabilities needed:
# docker run --cap-drop=ALL myapp
LABEL security.capabilities.drop="ALL" \
      security.capabilities.add="NET_BIND_SERVICE"
CMD ["nginx", "-g", "daemon off;"]
```

**Rationale:** By default, Docker grants containers 14 Linux capabilities including `NET_RAW` (packet spoofing), `SYS_CHROOT`, and `KILL`. `--cap-drop=ALL` removes them all, then `--cap-add` grants only what is needed. Most applications need zero capabilities. Web servers on ports >1024 need none. Only services binding to ports <1024 need `NET_BIND_SERVICE`. Documenting this in the Dockerfile ensures operators know the intended runtime security posture.

---

### HD006: No Package Manager in Runtime

- **Severity:** Medium
- **Tier:** Hardened
- **Description:** The final runtime image should not contain a package manager (apt, dpkg, yum, dnf, apk, pip, npm). Package managers enable attackers to install tools and download payloads.

**Bad example:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
# pip, apt, dpkg all available in runtime image
```

**Good example:**
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

FROM gcr.io/distroless/python3-debian12
WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY . .
ENV PYTHONPATH=/app/deps
CMD ["app.py"]
```

**Rationale:** A package manager in the runtime image lets an attacker run `apt-get install nmap ncat` or `pip install reverse-shell-tool` after gaining code execution. Distroless images, scratch images, and images with the package manager explicitly removed eliminate this attack vector. If distroless is not feasible, consider removing the package manager binaries in the final layer: `RUN apt-get purge -y --auto-remove apt dpkg && rm -rf /var/lib/dpkg`.

---

### HD007: COPY --chown Instead of RUN chown

- **Severity:** Low
- **Tier:** Hardened
- **Description:** Use `COPY --chown=user:group` to set file ownership at copy time instead of a separate `RUN chown` command. The separate RUN chown creates a duplicate layer with all the files, doubling the space consumed.

**Bad example:**
```dockerfile
COPY app/ /app/
RUN chown -R appuser:appuser /app/
```

**Good example:**
```dockerfile
COPY --chown=appuser:appuser app/ /app/
```

**Rationale:** `RUN chown -R` creates a new layer that contains a copy of every file with updated ownership metadata. If the app directory is 100MB, this adds another 100MB layer to the image. `COPY --chown` sets the ownership as part of the copy operation in a single layer, eliminating the duplication. This also applies to `ADD --chown` and, since Docker Engine 20.10+, `COPY --chmod` for setting permissions.

---

### HD008: Explicit EXPOSE Only

- **Severity:** Low
- **Tier:** Hardened
- **Description:** Only EXPOSE the ports that the application actually listens on. Do not omit EXPOSE (makes the image undocumented) and do not expose unnecessary ports.

**Bad example:**
```dockerfile
FROM nginx:1.25-alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/
# No EXPOSE - port is undocumented
CMD ["nginx", "-g", "daemon off;"]
```

**Good example:**
```dockerfile
FROM nginx:1.25-alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

**Rationale:** EXPOSE serves as documentation for which ports the container expects to receive traffic on. Missing EXPOSE makes the image opaque to operators and automated tooling. Exposing unnecessary ports (e.g., a debug port in production) signals that the port is intended to be open, increasing the attack surface. EXPOSE does not actually publish ports (that requires `-p` at runtime), but it communicates intent and integrates with orchestrator service discovery.

---

## STIG Tier Rules

> The STIG tier includes all Standard and Hardened tier rules plus the following.

### ST001: Rootfs-Builder Pattern

- **Severity:** High
- **Tier:** STIG
- **Description:** Use an `--installroot` pattern to build a minimal root filesystem in a builder stage, then copy only the rootfs into a scratch or minimal base. This produces a runtime image with only the exact packages needed and no package manager metadata.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.3
RUN microdnf install -y httpd mod_ssl && \
    microdnf clean all
COPY app/ /var/www/html/
EXPOSE 443
CMD ["httpd", "-D", "FOREGROUND"]
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:9.3 AS rootfs-builder
RUN dnf install -y \
        --installroot=/rootfs \
        --releasever=9 \
        --setopt=install_weak_deps=0 \
        --nodocs \
        httpd mod_ssl \
        glibc-minimal-langpack \
        tzdata && \
    dnf --installroot=/rootfs clean all && \
    rm -rf /rootfs/var/cache/dnf /rootfs/var/log/*

FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
COPY app/ /var/www/html/
EXPOSE 443
CMD ["httpd", "-D", "FOREGROUND"]
```

**Rationale:** The `--installroot` pattern creates a chroot-like filesystem tree containing only the requested packages and their dependencies. The full UBI base is used only during build for its package manager. The runtime image (UBI Micro) has no package manager, no shell, and only the files explicitly installed. This is the gold standard for minimal, STIG-compliant container images and typically produces images 60-80% smaller than standard approaches.

---

### ST002: Core Dumps Disabled

- **Severity:** Medium
- **Tier:** STIG
- **Description:** Core dumps must be disabled to prevent sensitive data (keys, passwords in memory) from being written to disk. Set `* hard core 0` in `/etc/security/limits.d/`.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
# No core dump limits set
# Default allows core dumps which may contain secrets
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
RUN echo '* hard core 0' > /etc/security/limits.d/99-core-dumps.conf
```

**Rationale:** Core dumps contain a snapshot of process memory at the time of a crash. This memory can include database credentials, encryption keys, session tokens, and PII. In containerized environments, core dumps may be written to a shared volume or host path where they can be accessed by other containers or host processes. DISA STIG V-230269 requires core dumps to be disabled. The `hard` limit prevents users from overriding it.

---

### ST003: Restrictive Umask 077

- **Severity:** Medium
- **Tier:** STIG
- **Description:** Set the default umask to 077 in `/etc/login.defs` so that newly created files are only readable by their owner. The default umask of 022 makes files world-readable.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
# Default umask 022: files created as 644 (world-readable)
# Directories created as 755 (world-traversable)
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
RUN sed -i 's/^UMASK.*/UMASK 077/' /etc/login.defs || \
    echo 'UMASK 077' >> /etc/login.defs
```

**Rationale:** A umask of 022 means files created by any process in the container are readable by all users. If an attacker gains access as a different user, they can read application config files, logs, and temporary files that may contain sensitive information. Umask 077 ensures files are created with permissions 600 (owner read/write only) and directories with 700 (owner only). DISA STIG V-230264 mandates umask 077 for all interactive users.

---

### ST004: No Empty Passwords

- **Severity:** Medium
- **Tier:** STIG
- **Description:** PAM configuration must not allow login with empty passwords. The `nullok` option must be removed from PAM authentication modules.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
# Default PAM config may include:
# auth sufficient pam_unix.so nullok
# This allows login with empty passwords
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
RUN sed -i 's/\s*nullok\s*/ /g' /etc/pam.d/system-auth /etc/pam.d/password-auth 2>/dev/null || true
```

**Rationale:** The `nullok` option in PAM configuration allows users with empty password fields in `/etc/shadow` to authenticate without a password. In containers, service accounts are sometimes created without passwords for convenience. If PAM allows `nullok`, any user who can reach the authentication stack (e.g., via SSH if installed, or su/sudo) can log in as these accounts. DISA STIG V-230380 requires removal of `nullok` from all PAM configurations.

---

### ST005: Max Concurrent Login Sessions

- **Severity:** Low
- **Tier:** STIG
- **Description:** Limit the maximum number of concurrent login sessions per user. This prevents a compromised account from spawning unlimited sessions for persistence or resource exhaustion.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
# No session limits configured
# Default: unlimited concurrent sessions
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
RUN echo '* hard maxlogins 10' > /etc/security/limits.d/99-maxlogins.conf
```

**Rationale:** Unlimited login sessions allow an attacker who has compromised credentials to maintain multiple footholds simultaneously, making detection and remediation harder. A limit of 10 concurrent sessions is sufficient for all legitimate use cases while constraining lateral movement. DISA STIG V-230346 requires that the operating system limit the number of concurrent sessions to 10 or fewer.

---

### ST006: GPG Checking for Packages

- **Severity:** Medium
- **Tier:** STIG
- **Description:** Package manager configuration must enforce GPG signature verification for all packages, including locally installed ones. This prevents installation of tampered or unsigned packages.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:9.3 AS rootfs-builder
# Default dnf.conf may have localpkg_gpgcheck=0
RUN dnf install -y --installroot=/rootfs mypackage.rpm
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:9.3 AS rootfs-builder
RUN echo 'localpkg_gpgcheck=1' >> /etc/dnf/dnf.conf && \
    dnf install -y \
        --installroot=/rootfs \
        --releasever=9 \
        --setopt=install_weak_deps=0 \
        httpd && \
    dnf --installroot=/rootfs clean all
```

**Rationale:** GPG signature verification ensures that packages have not been modified since they were signed by the distributor. Without `localpkg_gpgcheck=1`, locally provided RPMs bypass signature verification, allowing an attacker who can inject a malicious RPM into the build context to install arbitrary code. DISA STIG V-230264 requires GPG checking for all package sources. For Debian-based builds, ensure `apt-get` is configured with `Acquire::AllowInsecureRepositories "false"`.

---

### ST007: FIPS-Grade Crypto Config

- **Severity:** High
- **Tier:** STIG
- **Description:** OpenSSL must be configured to use only FIPS 140-2/140-3 approved algorithms. The crypto policy must restrict TLS to 1.2+ and disable weak ciphers (RC4, DES, MD5).

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
# Default OpenSSL config allows all algorithms
# TLS 1.0, 1.1 may be enabled
# Weak ciphers like RC4, 3DES may be available
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:9.3 AS rootfs-builder
RUN dnf install -y \
        --installroot=/rootfs \
        --releasever=9 \
        crypto-policies-scripts && \
    chroot /rootfs update-crypto-policies --set FIPS && \
    dnf --installroot=/rootfs clean all

FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
# Alternatively, configure OpenSSL directly:
# COPY fips-openssl.cnf /etc/pki/tls/openssl.cnf
```

**Rationale:** FIPS 140-2 compliance is required for federal government systems and is a best practice for any system handling sensitive data. Default OpenSSL configurations allow algorithms known to be weak (MD5, SHA-1 for signing, RC4, DES/3DES). RHEL 9's crypto-policies framework makes it straightforward to enforce FIPS system-wide. On non-RHEL systems, this requires a custom `openssl.cnf` with `[fips_sect]` configuration, `MinProtocol = TLSv1.2`, and an explicit list of approved ciphers.

---

### ST008: Clean Machine-ID

- **Severity:** Low
- **Tier:** STIG
- **Description:** The `/etc/machine-id` file must be empty or absent in the image so that each container instance generates a unique machine-id at boot. A baked-in machine-id means all containers from the image share the same identity.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
# /etc/machine-id contains the build host's machine-id
# All containers spawned from this image share it
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:9.3
COPY --from=rootfs-builder /rootfs/ /
RUN truncate -s 0 /etc/machine-id && \
    rm -f /var/lib/dbus/machine-id && \
    ln -s /etc/machine-id /var/lib/dbus/machine-id
```

**Rationale:** `/etc/machine-id` is used by systemd-journald, D-Bus, and other services to uniquely identify a host. When baked into a container image, every container instance shares the same machine-id, breaking assumptions about host uniqueness. This can cause log correlation issues, duplicate entries in monitoring systems, and in some cases security issues with DHCP clients or certificate generation. An empty file triggers automatic regeneration on first boot.

---

### ST009: Scanner Integration Stages

- **Severity:** Medium
- **Tier:** STIG
- **Description:** The Dockerfile should include build stages that run vulnerability scanners (Trivy, Grype, Snyk) against the final image. This builds security scanning into the image build pipeline rather than relying on external CI steps.

**Bad example:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/myapp /
CMD ["/myapp"]
# No scanning — vulnerabilities discovered later (or never)
```

**Good example:**
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

FROM gcr.io/distroless/static:nonroot AS runtime
COPY --from=builder /app/myapp /
CMD ["/myapp"]

FROM aquasec/trivy:latest AS scanner
COPY --from=runtime / /scan-target
RUN trivy rootfs --exit-code 1 --severity CRITICAL,HIGH --no-progress /scan-target

# Build the final image only if scanning passes:
# docker build --target runtime -t myapp .
# Or run the scanner stage to validate:
# docker build --target scanner -t myapp-scan .
```

**Rationale:** Integrating vulnerability scanning into the Dockerfile makes security checks reproducible and version-controlled alongside the application. Developers get immediate feedback when they introduce a vulnerable dependency. Using `--target` flags, CI pipelines can run the scanner stage as a gate and build the runtime stage as the deployable artifact. This is more reliable than relying on separate CI steps that can be skipped or misconfigured.

---

### ST010: Build Metadata LABELs

- **Severity:** Low
- **Tier:** STIG
- **Description:** Images must include LABEL instructions with build metadata: version, build date, VCS reference (git commit), maintainer, and description. This provides an audit trail for deployed images.

**Bad example:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
# No labels — image origin and version unknown
```

**Good example:**
```dockerfile
FROM python:3.12-slim
ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
LABEL org.opencontainers.image.title="myapp" \
      org.opencontainers.image.description="My application server" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.source="https://github.com/myorg/myapp" \
      org.opencontainers.image.vendor="MyOrg" \
      org.opencontainers.image.licenses="MIT"
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python", "app.py"]
```

**Rationale:** Without labels, there is no way to determine which version of the code a running container was built from, when it was built, or by whom. This makes incident response, rollback decisions, and compliance auditing extremely difficult. The OCI image spec defines standard label keys (`org.opencontainers.image.*`). Build arguments should be passed at build time: `docker build --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) --build-arg VCS_REF=$(git rev-parse HEAD) .`

---

### ST011: Minimal Langpack

- **Severity:** Low
- **Tier:** STIG
- **Description:** Use `glibc-minimal-langpack` instead of full locale packages. Full locale data adds significant size without benefit in containerized applications that typically use only `en_US.UTF-8` or `C.UTF-8`.

**Bad example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:9.3 AS rootfs-builder
RUN dnf install -y \
        --installroot=/rootfs \
        --releasever=9 \
        glibc-langpack-en \
        glibc-langpack-es \
        glibc-langpack-fr \
        httpd && \
    dnf --installroot=/rootfs clean all
```

**Good example:**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:9.3 AS rootfs-builder
RUN dnf install -y \
        --installroot=/rootfs \
        --releasever=9 \
        --setopt=install_weak_deps=0 \
        --nodocs \
        glibc-minimal-langpack \
        httpd && \
    dnf --installroot=/rootfs clean all
ENV LANG=C.UTF-8
```

**Rationale:** Full language packs can add 50-200MB of locale data to an image. Most containerized services only need a single locale for log output and string handling. `glibc-minimal-langpack` provides `C` and `C.UTF-8` locales, which are sufficient for almost all server applications. Setting `LANG=C.UTF-8` ensures consistent Unicode support. For Debian-based images, use `locales` with only the required locale generated via `locale-gen en_US.UTF-8`, or skip the locales package entirely and rely on `C.UTF-8`.

---

## Quick Reference Table

| Rule ID | Name | Severity | Tier |
|---------|------|----------|------|
| SD001 | Pinned base image tags | Critical | Standard |
| SD002 | Multi-stage builds | High | Standard |
| SD003 | Combined RUN layers | Medium | Standard |
| SD004 | Cache cleanup in same layer | Medium | Standard |
| SD005 | .dockerignore present | Medium | Standard |
| SD006 | No secrets in ENV/ARG/COPY | Critical | Standard |
| SD007 | HEALTHCHECK defined | Medium | Standard |
| SD008 | Specific COPY paths | Low | Standard |
| SD009 | Layer ordering for cache | Low | Standard |
| SD010 | Minimal base image selection | Medium | Standard |
| HD001 | Non-root USER | Critical | Hardened |
| HD002 | Read-only filesystem friendly | Medium | Hardened |
| HD003 | No shell in final image | Medium | Hardened |
| HD004 | Minimal packages | Medium | Hardened |
| HD005 | Drop all capabilities guidance | High | Hardened |
| HD006 | No package manager in runtime | Medium | Hardened |
| HD007 | COPY --chown instead of RUN chown | Low | Hardened |
| HD008 | Explicit EXPOSE only | Low | Hardened |
| ST001 | Rootfs-builder pattern | High | STIG |
| ST002 | Core dumps disabled | Medium | STIG |
| ST003 | Restrictive umask 077 | Medium | STIG |
| ST004 | No empty passwords | Medium | STIG |
| ST005 | Max concurrent login sessions | Low | STIG |
| ST006 | GPG checking for packages | Medium | STIG |
| ST007 | FIPS-grade crypto config | High | STIG |
| ST008 | Clean machine-id | Low | STIG |
| ST009 | Scanner integration stages | Medium | STIG |
| ST010 | Build metadata LABELs | Low | STIG |
| ST011 | Minimal langpack | Low | STIG |
