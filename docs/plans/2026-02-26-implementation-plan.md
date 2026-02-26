# secure-dockerfile Plugin Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Claude Code skills plugin with three slash commands (`audit`, `create`, `fix`) that help users create, analyze, and harden Dockerfiles across three security tiers.

**Architecture:** Pure skills plugin — markdown skill files + knowledge base documents. No code, no runtime dependencies. Skills read knowledge files to build Claude's Dockerfile expertise, then guide it through structured workflows.

**Tech Stack:** Markdown (YAML frontmatter), JSON (plugin manifest, SARIF schema)

---

### Task 1: Plugin Manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

**Step 1: Create the plugin manifest**

```json
{
  "name": "secure-dockerfile",
  "description": "Dockerfile security auditing, hardening, and generation — reduce image size, eliminate vulnerabilities, follow best practices across three security tiers",
  "version": "0.1.0",
  "author": {
    "name": "Brandon Caldwell",
    "url": "https://github.com/brandonrc"
  },
  "repository": "https://github.com/brandonrc/secure-that-dockerfile",
  "license": "MIT",
  "keywords": ["dockerfile", "security", "containers", "hardening", "stig", "devops", "docker"]
}
```

**Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add plugin manifest"
```

---

### Task 2: Security Tiers Knowledge Base

**Files:**
- Create: `knowledge/security-tiers.md`

This is the core reference document. Every skill reads it. It defines every rule with an ID, severity, tier, description, bad example, good example, and rationale.

**Step 1: Write `knowledge/security-tiers.md`**

Structure:

```markdown
# Security Tiers

## Rule Format
Each rule has: ID, Severity, Tier, Description, Bad Example, Good Example, Rationale

## Standard Tier Rules
- SD001: Pinned base image tags (Critical)
- SD002: Multi-stage builds (High)
- SD003: Combined RUN layers (Medium)
- SD004: Cache cleanup in same layer (Medium)
- SD005: .dockerignore present (Medium)
- SD006: No secrets in ENV/ARG/COPY (Critical)
- SD007: HEALTHCHECK defined (Medium)
- SD008: Specific COPY paths (Low)
- SD009: Layer ordering for cache (Low)
- SD010: Minimal base image selection (Medium)

## Hardened Tier Rules (includes Standard)
- HD001: Non-root USER (Critical)
- HD002: Read-only filesystem friendly (Medium)
- HD003: No shell in final image (Medium)
- HD004: Minimal packages (--no-install-recommends) (Medium)
- HD005: Drop all capabilities guidance (High)
- HD006: No package manager in runtime (Medium)
- HD007: COPY --chown instead of RUN chown (Low)
- HD008: Explicit EXPOSE only (Low)

## STIG Tier Rules (includes Hardened)
- ST001: Rootfs-builder pattern (High)
- ST002: Core dumps disabled (Medium)
- ST003: Restrictive umask 077 (Medium)
- ST004: No empty passwords (Medium)
- ST005: Max concurrent login sessions (Low)
- ST006: GPG checking for packages (Medium)
- ST007: FIPS-grade crypto config (High)
- ST008: Clean machine-id (Low)
- ST009: Scanner integration stages (Medium)
- ST010: Build metadata LABELs (Low)
- ST011: Minimal langpack (Low)
```

Each rule gets a full entry with bad/good examples and rationale. This is the largest single file — expect ~400-500 lines.

**Step 2: Commit**

```bash
git add knowledge/security-tiers.md
git commit -m "feat: add security tiers knowledge base with rule definitions"
```

---

### Task 3: Best Practices Knowledge Base

**Files:**
- Create: `knowledge/best-practices.md`

**Step 1: Write `knowledge/best-practices.md`**

Universal Dockerfile guidance covering:
- Layer hygiene (combining RUN, ordering, cleanup)
- Build cache optimization (dependency layers before source)
- Secret management (BuildKit secrets, multi-stage isolation)
- .dockerignore patterns
- HEALTHCHECK design
- ENTRYPOINT vs CMD
- ARG vs ENV scoping
- Reproducible builds (pinned versions, checksums)

**Step 2: Commit**

```bash
git add knowledge/best-practices.md
git commit -m "feat: add best practices knowledge base"
```

---

### Task 4: Multi-Stage Builds Knowledge Base

**Files:**
- Create: `knowledge/multi-stage.md`

**Step 1: Write `knowledge/multi-stage.md`**

Patterns for multi-stage builds:
- Basic builder + runtime pattern
- Dependency caching stage (cargo-chef, pip wheel, npm ci)
- Rootfs-builder pattern (--installroot for minimal runtime)
- Scanner integration stage
- Asset compilation stage (frontend builds)
- Stage naming conventions
- COPY --from references
- When to use scratch vs distroless vs minimal base

**Step 2: Commit**

```bash
git add knowledge/multi-stage.md
git commit -m "feat: add multi-stage builds knowledge base"
```

---

### Task 5: Base Images Knowledge Base

**Files:**
- Create: `knowledge/base-images.md`

**Step 1: Write `knowledge/base-images.md`**

Comparison of base images with size/security/compatibility trade-offs:
- **scratch** — zero base, static binaries only
- **distroless** — Google's minimal images, no shell/package manager
- **Alpine** — musl-based, ~5MB, wide ecosystem
- **Debian slim** — glibc, ~80MB, broad compatibility
- **UBI 9 Micro** — Red Hat, FIPS-capable, enterprise support
- **UBI 9 Minimal** — UBI with microdnf, slightly larger
- **Chainguard** — signed, SBOM-included, minimal
- When to pick which, and for what tier

**Step 2: Commit**

```bash
git add knowledge/base-images.md
git commit -m "feat: add base images knowledge base"
```

---

### Task 6: SARIF Format Knowledge Base

**Files:**
- Create: `knowledge/sarif-format.md`

**Step 1: Write `knowledge/sarif-format.md`**

SARIF 2.1.0 template that the audit skill uses to produce output:
- Schema reference
- Tool definition block (secure-dockerfile as the tool)
- Rule definitions mapped from security-tiers.md rule IDs
- Result template with level mapping (Critical/High → error, Medium → warning, Low/Info → note)
- Physical location format pointing to Dockerfile lines
- Complete example SARIF output for a sample audit

**Step 2: Commit**

```bash
git add knowledge/sarif-format.md
git commit -m "feat: add SARIF format template for audit output"
```

---

### Task 7: Rust UBI9 Example

**Files:**
- Create: `knowledge/examples/rust-ubi9.md`

**Step 1: Write `knowledge/examples/rust-ubi9.md`**

The user's example Dockerfile as a fully annotated reference pattern. Include:
- The complete Dockerfile
- Annotations explaining each stage and why it exists
- Which tier rules each section satisfies
- Size impact notes

**Step 2: Commit**

```bash
git add knowledge/examples/rust-ubi9.md
git commit -m "feat: add Rust UBI9 STIG example as reference pattern"
```

---

### Task 8: Audit Skill

**Files:**
- Create: `skills/audit/SKILL.md`

**Step 1: Write the audit skill**

YAML frontmatter:
```yaml
---
name: audit
description: Analyze a Dockerfile for security issues, size bloat, and best practice violations. Supports --report (markdown) and --sarif output formats.
argument-hint: [path/to/Dockerfile] [--report] [--sarif]
---
```

The skill body instructs Claude to:
1. Parse arguments for Dockerfile path and output flags
2. If no path given, find Dockerfile in project root (Glob for `**/Dockerfile*`)
3. Read the Dockerfile
4. Read `knowledge/security-tiers.md` and `knowledge/best-practices.md` from the plugin directory (`${CLAUDE_PLUGIN_ROOT}/knowledge/`)
5. Analyze line-by-line against every rule
6. Score: start at 100, deduct per finding (Critical: -15, High: -10, Medium: -5, Low: -2)
7. Determine current tier (meets Standard? Hardened? STIG?)
8. Output structured findings grouped by severity
9. If `--report` flag: write `dockerfile-audit.md` to project root
10. If `--sarif` flag: read `knowledge/sarif-format.md` template and write `dockerfile-audit.sarif`

**Step 2: Commit**

```bash
git add skills/audit/SKILL.md
git commit -m "feat: add audit skill for Dockerfile analysis"
```

---

### Task 9: Create Skill

**Files:**
- Create: `skills/create/SKILL.md`

**Step 1: Write the create skill**

YAML frontmatter:
```yaml
---
name: create
description: Generate a new hardened Dockerfile from scratch by auto-detecting your project's language and framework. Supports security tiers (standard, hardened, stig).
argument-hint: [--tier standard|hardened|stig]
---
```

The skill body instructs Claude to:
1. Parse optional `--tier` argument (default: prompt user)
2. Auto-detect project language by scanning for manifest files (Cargo.toml, package.json, go.mod, requirements.txt, pyproject.toml, pom.xml, build.gradle, Gemfile, mix.exs, CMakeLists.txt)
3. Present detection to user for confirmation
4. Read `knowledge/security-tiers.md`, `knowledge/multi-stage.md`, `knowledge/base-images.md` from `${CLAUDE_PLUGIN_ROOT}/knowledge/`
5. If tier not specified, ask user which tier (with descriptions)
6. Generate multi-stage Dockerfile following tier rules
7. Check for `.dockerignore` — generate if missing
8. Explain each stage with inline comments
9. Write Dockerfile to project root

**Step 2: Commit**

```bash
git add skills/create/SKILL.md
git commit -m "feat: add create skill for Dockerfile generation"
```

---

### Task 10: Fix Skill

**Files:**
- Create: `skills/fix/SKILL.md`

**Step 1: Write the fix skill**

YAML frontmatter:
```yaml
---
name: fix
description: Rewrite an existing Dockerfile to fix security issues and follow best practices. Runs audit first, then restructures the Dockerfile to meet the target tier.
argument-hint: [path/to/Dockerfile] [--tier standard|hardened|stig]
---
```

The skill body instructs Claude to:
1. Parse arguments for Dockerfile path and target tier
2. Find and read the existing Dockerfile
3. Run the audit analysis internally (same logic as audit skill — read knowledge files, analyze rules)
4. Present findings to user
5. Ask which tier to target (if not specified)
6. Read `knowledge/multi-stage.md` and `knowledge/base-images.md`
7. Rewrite the Dockerfile:
   - Preserve all app-specific logic (what the app needs)
   - Restructure for the target tier (how it's built)
   - Add stages as needed (builder, deps, rootfs-builder, runtime)
8. Show what changed and why (summary, not a raw diff)
9. Write the updated Dockerfile

**Step 2: Commit**

```bash
git add skills/fix/SKILL.md
git commit -m "feat: add fix skill for Dockerfile hardening"
```

---

### Task 11: README

**Files:**
- Create: `README.md`

**Step 1: Write the README**

Covers:
- What the plugin does (one paragraph)
- Installation: how to install from the git repo in Claude Code
- Commands: `/secure-dockerfile:audit`, `/secure-dockerfile:create`, `/secure-dockerfile:fix` with usage examples
- Security tiers: brief description of each
- Output formats: conversation, markdown, SARIF
- Contributing / License

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with installation and usage guide"
```

---

### Task 12: LICENSE

**Files:**
- Create: `LICENSE`

**Step 1: Write MIT license file**

Standard MIT license with Brandon Caldwell as copyright holder, 2026.

**Step 2: Commit**

```bash
git add LICENSE
git commit -m "chore: add MIT license"
```

---

### Task 13: Integration Test

**Step 1: Create a test Dockerfile to audit**

Create a deliberately bad `tests/bad-dockerfile/Dockerfile` with common issues:
- `FROM ubuntu:latest`
- `ENV DB_PASSWORD=secret`
- `RUN apt-get update` then separate `RUN apt-get install`
- No USER directive
- No HEALTHCHECK
- `COPY . .`
- No multi-stage build

**Step 2: Manually test audit skill**

In Claude Code, run `/secure-dockerfile:audit tests/bad-dockerfile/Dockerfile` and verify it catches all issues.

**Step 3: Commit test fixtures**

```bash
git add tests/
git commit -m "test: add deliberately bad Dockerfile for integration testing"
```

---

## Dependency Order

```
Task 1 (manifest) — no dependencies, everything else needs this
Task 2-7 (knowledge) — independent of each other, can be parallel
Task 8-10 (skills) — depend on knowledge files (Tasks 2-7)
Task 11-12 (README, LICENSE) — depend on skills existing
Task 13 (test) — depends on everything
```

## Parallel Opportunities

- Tasks 2, 3, 4, 5, 6, 7 are fully independent (all knowledge files)
- Tasks 8, 9, 10 are independent of each other (but need Tasks 2-7 done first)
- Tasks 11, 12 are independent of each other
