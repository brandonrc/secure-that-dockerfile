# Base Images Reference

Comparison of container base images with size, security, and compatibility trade-offs. Use this document to select the right base image for the target security tier and application requirements.

---

## scratch

- **Image:** `scratch` (built-in Docker keyword)
- **Size:** 0 bytes
- **libc:** None
- **Shell:** None
- **Package Manager:** None
- **FIPS:** N/A (no OS layer)

**Use case:** Statically-linked binaries only. Go with `CGO_ENABLED=0`, Rust with `--target x86_64-unknown-linux-musl`, or any binary compiled with musl and fully static linking.

**Security:** Maximum possible. No operating system, no shell, no tools, no package manager, no users, no CVEs. The image contains only what you `COPY` into it.

**Downsides:**
- No shell — cannot `docker exec` into the container for debugging
- No libc — the binary must be fully statically linked
- No DNS resolution unless the binary handles DNS itself (Go's net package with CGO_ENABLED=0 uses a pure Go resolver; Rust needs a static DNS solution)
- No CA certificates — TLS connections fail unless `/etc/ssl/certs/ca-certificates.crt` is copied from a builder stage
- No `/tmp`, no `/etc/passwd`, no timezone data unless explicitly added
- Error messages from the container runtime may be cryptic with no OS context

**Tier fit:** Hardened, STIG

**Example:**
```dockerfile
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app .

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

---

## Google Distroless

- **Images:**
  - `gcr.io/distroless/static-debian12` — no libc, for fully static binaries
  - `gcr.io/distroless/base-debian12` — glibc, for dynamically-linked binaries
  - `gcr.io/distroless/cc-debian12` — glibc + libstdc++, for C++ binaries
  - Language-specific variants: `gcr.io/distroless/java21-debian12`, `gcr.io/distroless/python3-debian12`, `gcr.io/distroless/nodejs22-debian12`
- **Size:** ~2 MB (static) to ~20 MB (language-specific variants)
- **libc:** None (static), glibc (base/cc/language variants)
- **Shell:** None (use `:debug` tag to get a busybox shell for troubleshooting)
- **Package Manager:** None
- **FIPS:** Not natively — depends on application-level crypto configuration

**Use case:** Applications that need glibc or a language runtime but nothing else. Suitable for Go, Rust, Java, Python, and Node.js applications where you want minimal attack surface without going all the way to scratch.

**Security:** No shell, no package manager, no utility programs. The attack surface is limited to glibc and the specific runtime libraries included. Google maintains and patches these images regularly.

**Downsides:**
- Harder to debug in production — no shell means no `docker exec` troubleshooting (use the `:debug` tag in non-production environments)
- Limited to glibc-based applications — musl-compiled binaries may not work with `base` or `cc` variants
- No ability to install additional OS packages at runtime
- Timezone data and CA certificates are included, but other OS-level files may be missing

**Tier fit:** Hardened, STIG

---

## Alpine

- **Image:** `alpine:3.21`
- **Size:** ~5 MB
- **libc:** musl
- **Shell:** `/bin/sh` (BusyBox)
- **Package Manager:** `apk`
- **FIPS:** Not natively — musl's crypto has no FIPS certification

**Use case:** General-purpose minimal image with a full package ecosystem. Good for applications where you need a small image but also need to install runtime dependencies or debugging tools.

**Security:** Small attack surface due to size. BusyBox provides minimal utilities. The `apk` package manager uses a strong signature verification system. Fewer pre-installed packages means fewer CVEs compared to Debian or Ubuntu.

**Downsides:**
- musl libc can cause compatibility issues with software expecting glibc:
  - DNS resolution behavior differs (no `search` domain fallback by default in some configurations)
  - Locale support is limited (no full `glibc-langpack`, `LC_ALL` behavior may differ)
  - Some C libraries with glibc-specific extensions will not compile or run correctly
  - NSS (Name Service Switch) is not supported — LDAP/NIS user lookups do not work
- Python packages with C extensions may need to be compiled from source (no manylinux wheel support with musl)
- Debugging tools must be installed via `apk add` — they are not present by default
- Some commercial software explicitly does not support musl-based distributions

**Tier fit:** Standard, Hardened

**Example:**
```dockerfile
FROM alpine:3.21
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
ENTRYPOINT ["/app"]
```

---

## Debian Slim

- **Image:** `debian:bookworm-slim`
- **Size:** ~80 MB
- **libc:** glibc
- **Shell:** `/bin/bash`
- **Package Manager:** `apt`
- **FIPS:** Not natively — available via configuration but not certified

**Use case:** Broad glibc compatibility with a large package repository. Suitable for applications that depend on glibc behavior, need packages from the Debian ecosystem, or encounter musl compatibility issues on Alpine.

**Security:** Moderate attack surface. The `slim` variant removes documentation, man pages, and some less-used files from the full Debian image. Debian has a well-established security team with regular patch cadence. However, the image ships with more packages than typically needed.

**Downsides:**
- Significantly larger than Alpine or distroless (~16x Alpine)
- Includes more pre-installed packages than most applications need, increasing CVE exposure
- `apt` package manager is present in the image — consider removing or using a multi-stage approach
- Not minimal enough for Hardened or STIG tiers without significant additional stripping

**Tier fit:** Standard

**Example:**
```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app /app
RUN useradd --create-home --shell /usr/sbin/nologin appuser
USER appuser
ENTRYPOINT ["/app"]
```

---

## UBI 9 Micro

- **Image:** `registry.access.redhat.com/ubi9/ubi-micro:9.7`
- **Size:** ~30 MB
- **libc:** glibc
- **Shell:** None (no `/bin/sh` by default)
- **Package Manager:** None in the final image (use rootfs-builder pattern to install packages during build)
- **FIPS:** Yes — Red Hat's OpenSSL is FIPS 140-2/140-3 validated when configured

**Use case:** RHEL-compatible minimal runtime for enterprise and government environments. Ideal when FIPS compliance, STIG hardening, or RHEL-based support contracts are required. Best used with the rootfs-builder pattern where `ubi9/ubi` or `ubi9/ubi-minimal` serves as the build stage and installs packages into `ubi-micro` via `--installroot`.

**Security:** No package manager in the runtime image. No shell by default. Red Hat's security team maintains patch cadence. STIG-ready when combined with hardening controls. FIPS-validated crypto libraries are included.

**Downsides:**
- Larger than distroless or Chainguard static images
- RHEL-specific packages — not all Debian/Alpine packages are available
- The rootfs-builder pattern adds complexity to the Dockerfile
- Requires Red Hat registry access (free for UBI images, but the registry URL is long)
- Fewer community examples compared to Alpine or Debian

**Tier fit:** Hardened, STIG (especially with rootfs-builder pattern)

**Example (rootfs-builder pattern):**
```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.7 AS rootfs-builder
RUN microdnf install --installroot /rootfs --releasever 9 --setopt install_weak_deps=0 --nodocs -y \
    glibc-minimal-langpack \
    ca-certificates \
    tzdata \
    && microdnf --installroot /rootfs clean all \
    && rm -rf /rootfs/var/cache/* /rootfs/var/log/*

FROM registry.access.redhat.com/ubi9/ubi-micro:9.7
COPY --from=rootfs-builder /rootfs/ /
COPY --from=builder /app /app
USER 1001
ENTRYPOINT ["/app"]
```

---

## UBI 9 Minimal

- **Image:** `registry.access.redhat.com/ubi9/ubi-minimal:9.7`
- **Size:** ~90 MB
- **libc:** glibc
- **Shell:** `/bin/sh` (via `microdnf` dependencies)
- **Package Manager:** `microdnf`
- **FIPS:** Yes — same FIPS-validated OpenSSL as UBI Micro

**Use case:** When you need a Red Hat-compatible image with the ability to install packages at build time without the rootfs-builder pattern. Also used as the builder stage in rootfs-builder workflows. Suitable when runtime package management is acceptable.

**Security:** Smaller than full UBI (`ubi9/ubi` is ~200 MB). Has `microdnf` instead of full `dnf`, limiting but not eliminating package management capability. The presence of a package manager in a runtime image is a security concern — an attacker who gains code execution can install tools.

**Downsides:**
- Package manager (`microdnf`) is present in the runtime image, which is less secure than UBI Micro
- Larger than UBI Micro, Alpine, or distroless
- Not suitable for STIG tier as a runtime image due to the package manager
- Better suited as a build stage than a runtime stage

**Tier fit:** Standard, Hardened (primarily as a build/builder stage, not as the runtime image)

---

## Chainguard

- **Images:**
  - `cgr.dev/chainguard/static` — no libc, for fully static binaries
  - `cgr.dev/chainguard/glibc-dynamic` — glibc, for dynamically-linked binaries
  - Language-specific variants: `cgr.dev/chainguard/go`, `cgr.dev/chainguard/python`, `cgr.dev/chainguard/node`, `cgr.dev/chainguard/jre`
- **Size:** ~2 MB (static) to ~15 MB (language-specific variants)
- **libc:** None (static), glibc (glibc-dynamic/language variants)
- **Shell:** None (use `:latest-dev` tag for shell access during development)
- **Package Manager:** None (`:latest-dev` tag includes `apk`)
- **FIPS:** Available on paid tier (Chainguard FIPS images)

**Use case:** Supply chain security as a first-class concern. All Chainguard images include SBOMs (Software Bill of Materials), are signed with Sigstore/cosign, and are rebuilt daily to minimize time-to-patch. Suitable when you need verifiable provenance and minimal CVE exposure.

**Security:** Daily rebuilds mean known CVEs are patched faster than most other base images. Zero known CVEs is the target state. Sigstore signatures provide cryptographic verification of image provenance. SBOMs enable automated vulnerability scanning and compliance auditing.

**Downsides:**
- Some images and features (FIPS, extended support, SLA) require a paid Chainguard subscription
- Smaller ecosystem and community compared to Alpine or Debian — fewer tutorials and troubleshooting resources
- The `:latest` tag is the hardened image (no shell); `:latest-dev` adds a shell, which can cause confusion
- Image names and tags differ from Docker Hub conventions

**Tier fit:** Hardened, STIG

---

## Comparison Table

| Image | Size | libc | Shell | Pkg Manager | FIPS | Best For |
|---|---|---|---|---|---|---|
| `scratch` | 0 B | None | No | No | N/A | Static Go/Rust binaries |
| Distroless static | ~2 MB | None | No | No | No | Static binaries with CA certs/tzdata |
| Distroless base | ~20 MB | glibc | No | No | No | Dynamic binaries needing glibc |
| Alpine 3.21 | ~5 MB | musl | Yes | apk | No | General-purpose minimal image |
| Debian bookworm-slim | ~80 MB | glibc | Yes | apt | No | Broad glibc compatibility |
| UBI 9 Micro | ~30 MB | glibc | No | No | Yes | FIPS/STIG enterprise runtime |
| UBI 9 Minimal | ~90 MB | glibc | Yes | microdnf | Yes | RHEL builder stages |
| Chainguard static | ~2 MB | None | No | No | Paid | Supply chain security, signed images |
| Chainguard glibc-dynamic | ~15 MB | glibc | No | No | Paid | Dynamic binaries with provenance |

---

## Selection Guide

Use the following decision tree to select a base image based on application requirements and target security tier.

**"I have a static Go binary"**
Use `scratch` or `cgr.dev/chainguard/static`. Both are near-zero size with no OS layer. Chainguard adds CA certs, tzdata, and a non-root user by default. With `scratch`, you must copy those in yourself.

**"I need libc but nothing else"**
Use `gcr.io/distroless/base-debian12` or `cgr.dev/chainguard/glibc-dynamic`. Both provide glibc without a shell or package manager. Chainguard offers signed images and SBOMs. Distroless has wider community adoption.

**"I need a small general-purpose image"**
Use `alpine:3.21`. It provides a full package manager, a shell for debugging, and a ~5 MB footprint. Verify your application works with musl libc before committing to Alpine.

**"I need broad compatibility"**
Use `debian:bookworm-slim`. It provides glibc, bash, and the full Debian package ecosystem. Use this when Alpine's musl causes compatibility issues or when your dependencies require glibc-specific behavior.

**"I need FIPS/STIG compliance"**
Use `registry.access.redhat.com/ubi9/ubi-micro:9.7` with the rootfs-builder pattern. UBI Micro provides FIPS-validated crypto, Red Hat security patching, and a STIG-ready baseline. Use `ubi9/ubi-minimal` as the builder stage to install packages via `--installroot`.

**"I need supply chain guarantees"**
Use `cgr.dev/chainguard/static` or the appropriate Chainguard language variant. Chainguard images are signed with Sigstore, include SBOMs, and target zero known CVEs through daily rebuilds.

**"I am building a multi-stage Dockerfile and need a builder image"**
Use the language-specific official image (e.g., `golang:1.23`, `rust:1.84`, `node:22`) or `ubi9/ubi-minimal` for the builder stage. The builder image does not appear in the final image, so its size and attack surface matter less. Focus base image selection on the runtime stage.

---

## Tier Compatibility Matrix

| Image | Standard | Hardened | STIG |
|---|---|---|---|
| `scratch` | Yes | Yes | Yes |
| Distroless (static/base/cc) | Yes | Yes | Yes |
| Alpine | Yes | Yes (remove apk in final stage) | Not recommended |
| Debian slim | Yes | Not recommended | No |
| UBI 9 Micro | Yes | Yes | Yes |
| UBI 9 Minimal | Yes (builder only) | Yes (builder only) | No (as runtime) |
| Chainguard | Yes | Yes | Yes |

**Notes on tier compatibility:**
- For **Hardened** tier, the runtime image should not contain a shell or package manager. Alpine can qualify if `apk` is removed in the final stage, but this is fragile and distroless/Chainguard/UBI Micro are preferred.
- For **STIG** tier, FIPS-validated crypto is typically required, which limits practical choices to UBI 9 Micro (with rootfs-builder) or Chainguard FIPS images (paid tier).
- Any image can serve as a builder stage regardless of tier — only the runtime (final) stage determines tier compliance.
