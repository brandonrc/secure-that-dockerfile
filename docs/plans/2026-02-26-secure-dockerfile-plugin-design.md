# secure-dockerfile Plugin Design

**Date:** 2026-02-26
**Status:** Approved
**Architecture:** Pure Skills Plugin (no MCP server, no runtime dependencies)

## Overview

A Claude Code plugin that helps users create, audit, and harden Dockerfiles. It ships as markdown skill files with an embedded knowledge base — zero dependencies, leverages Claude's native Dockerfile reasoning.

## Plugin Structure

```
secure-dockerfile/
├── .claude-plugin/
│   └── plugin.json           # Plugin manifest
├── skills/
│   ├── audit/
│   │   └── SKILL.md          # /secure-dockerfile:audit
│   ├── create/
│   │   └── SKILL.md          # /secure-dockerfile:create
│   └── fix/
│       └── SKILL.md          # /secure-dockerfile:fix
├── knowledge/
│   ├── best-practices.md     # Universal Dockerfile rules
│   ├── multi-stage.md        # Multi-stage build patterns
│   ├── security-tiers.md     # Standard / Hardened / STIG tier definitions
│   ├── base-images.md        # Base image guidance (UBI, Alpine, Distroless, etc.)
│   ├── sarif-format.md       # SARIF output template/schema
│   └── examples/
│       └── rust-ubi9.md      # Reference pattern: Rust on UBI9 with STIG
├── README.md
└── LICENSE
```

## Slash Commands

### `secure-dockerfile:audit`

**Purpose:** Analyze an existing Dockerfile and report security issues, size bloat, and best practice violations.

**Workflow:**

1. Find the Dockerfile in the project (or accept a path argument)
2. Read and analyze against the knowledge base
3. Categorize findings by severity: Critical / High / Medium / Low / Info
4. Group findings into: Security, Size, Best Practices
5. Output findings inline in conversation (default)
6. Optionally write `dockerfile-audit.md` report (user passes `--report`)
7. Optionally write `dockerfile-audit.sarif` (user passes `--sarif`)

**Findings it detects:**

- Running as root
- No HEALTHCHECK
- Secrets in ENV/ARG
- `apt-get update` without `apt-get install` in same RUN
- Missing cache cleanup (`--no-cache`, `dnf clean all`)
- No multi-stage build
- Unpinned base image tags
- Missing `.dockerignore`
- Unnecessary packages in runtime image
- COPY . . instead of specific paths
- Poor layer ordering (source before dependencies)
- Writable filesystem concerns
- Missing capability drops guidance

**Output formats:**

- **Conversation (default):** Structured findings with severity, line references, and remediation
- **Markdown report (`--report`):** Same content written to `dockerfile-audit.md`
- **SARIF (`--sarif`):** SARIF 2.1.0 JSON for CI/CD integration (GitHub Code Scanning, VS Code SARIF Viewer)

**Scoring:** Each Dockerfile gets a score out of 100 with the current tier level indicated.

### `secure-dockerfile:create`

**Purpose:** Generate a new hardened Dockerfile from scratch.

**Workflow:**

1. Auto-detect project: scan for Cargo.toml, package.json, go.mod, requirements.txt, pom.xml, etc.
2. Confirm detected language/framework with user
3. Ask which security tier: standard / hardened / stig
4. Generate multi-stage Dockerfile following tier rules
5. Generate `.dockerignore` if missing
6. Explain what each stage does

### `secure-dockerfile:fix`

**Purpose:** Take an existing Dockerfile and rewrite it to fix issues.

**Workflow:**

1. Run audit internally (same analysis logic)
2. Present findings to user
3. Ask which tier to target
4. Rewrite the Dockerfile, preserving app-specific logic but restructuring for security/size
5. Show a diff of what changed and why

## Security Tiers

### Standard

The baseline every Dockerfile should meet:

- Multi-stage builds — separate build from runtime
- Pinned base image tags — `node:20.11-slim` not `node:latest`
- Minimal base images — prefer `-slim`, `-alpine`, `ubi-minimal`
- Combined RUN layers — `apt-get update && apt-get install` in one layer
- Cache cleanup — `apt-get clean`, `dnf clean all`, `rm -rf /var/cache`
- `.dockerignore` present and sensible
- No secrets in image — no ENV/ARG for passwords, no COPY of `.env`
- HEALTHCHECK defined
- Specific COPY instead of `COPY . .` where possible
- Layer ordering — dependencies before source code for cache efficiency

### Hardened

Everything in Standard, plus:

- Non-root USER — create and switch to unprivileged user
- Read-only filesystem friendly (no writes to `/tmp` without explicit mount)
- No shell in final image where possible (distroless/scratch patterns)
- Minimal packages — `--no-install-recommends`, `install_weak_deps=0`
- Drop all capabilities guidance (runtime `--cap-drop=ALL`)
- No package manager in runtime image
- `COPY --chown` instead of `RUN chown`
- Explicit EXPOSE for documented ports only

### STIG

Everything in Hardened, plus:

- Rootfs-builder pattern — `--installroot` to build minimal rootfs
- STIG hardening controls:
  - Core dumps disabled (`* hard core 0`)
  - Restrictive umask (077)
  - No empty passwords (`nullok` removed)
  - Max concurrent login sessions
  - GPG checking for packages (`localpkg_gpgcheck=1`)
- FIPS-grade crypto — OpenSSL FIPS configuration
- Clean machine-id (regenerated at runtime)
- Scanner integration stages — trivy/grype baked into build pipeline
- Audit trail — LABEL with build metadata, version info
- Minimal langpack — `glibc-minimal-langpack` over full locales

## Knowledge Base

The `knowledge/` directory contains reference material that skills read to build context:

- **best-practices.md** — Universal rules (layer hygiene, caching, ordering, secrets)
- **multi-stage.md** — Patterns for multi-stage builds (builder, deps, rootfs-builder, scanners, runtime)
- **security-tiers.md** — Full tier definitions with rule IDs and remediation examples
- **base-images.md** — Comparison of base images (UBI9, Alpine, Distroless, Chainguard, scratch) with size/security trade-offs
- **sarif-format.md** — SARIF 2.1.0 template for audit output
- **examples/rust-ubi9.md** — Complete reference implementation: Rust on UBI9 Micro with STIG hardening

## Audit Output Examples

### Conversation Output

```
## Dockerfile Audit Results

**Overall Score: 62/100** (Standard tier)

### Critical (2)
- **Line 1:** Base image `ubuntu:latest` uses unpinned tag — use `ubuntu:24.04`
- **Line 14:** `ENV DB_PASSWORD=secret123` — secret exposed in image layer

### High (3)
- No non-root USER directive — container runs as root
- No HEALTHCHECK defined
- No multi-stage build — build tools present in runtime image

### Medium (4)
- Line 8: `RUN apt-get update` separate from `apt-get install`
...

### Recommendations
To reach **Hardened** tier, address all Critical + High findings.
```

### SARIF Output

Standard SARIF 2.1.0 JSON where each finding maps to:
- `result.ruleId` — rule identifier (e.g., `SD001-unpinned-tag`)
- `result.level` — error/warning/note
- `result.message.text` — human-readable description
- `result.locations[].physicalLocation` — Dockerfile path and line number

## Design Decisions

1. **Pure skills, no MCP server** — Claude already excels at reading and reasoning about Dockerfiles. No need for external tools when the knowledge base makes Claude an expert.

2. **Language-agnostic knowledge base** — Focus on universal Dockerfile principles that apply to any stack. Language-specific patterns emerge naturally from project detection.

3. **Tiered approach over binary secure/insecure** — Security is a spectrum. Not every project needs STIG compliance. Three tiers let users pick the right level.

4. **Knowledge separate from skills** — Skills define workflows (what Claude does). Knowledge defines expertise (what Claude knows). This separation makes both easier to maintain and extend.

5. **Auto-detect + confirm for create** — Scanning the project gives Claude context, confirming with the user prevents wrong assumptions. Best of both worlds.
