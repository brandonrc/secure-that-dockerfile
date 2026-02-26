# secure-dockerfile

A Claude Code plugin that helps you create, audit, and harden Dockerfiles. Reduce image size, eliminate vulnerabilities, and follow best practices across three security tiers.

## Installation

```bash
# Step 1: Add the marketplace
/plugin marketplace add brandonrc/secure-that-dockerfile

# Step 2: Open plugin manager and install
/plugin
```

Then go to the **Discover** tab, find **secure-dockerfile**, and install it.

## Commands

### `/secure-dockerfile:audit`

Analyze an existing Dockerfile for security issues, size bloat, and best practice violations.

```
/secure-dockerfile:audit                          # auto-find Dockerfile
/secure-dockerfile:audit path/to/Dockerfile       # specific file
/secure-dockerfile:audit --report                 # also write dockerfile-audit.md
/secure-dockerfile:audit --sarif                  # also write dockerfile-audit.sarif
/secure-dockerfile:audit --report --sarif         # both output formats
```

Checks 29 rules across three tiers. Outputs a score (0-100), severity-grouped findings with line numbers, and remediation guidance.

### `/secure-dockerfile:create`

Generate a new hardened Dockerfile by auto-detecting your project's language and framework.

```
/secure-dockerfile:create                         # interactive (detect + ask tier)
/secure-dockerfile:create --tier hardened          # skip tier selection
```

Supports: Rust, Go, Node.js, Python, Java/Kotlin, Ruby, Elixir, C/C++, .NET.

Also generates a `.dockerignore` if one doesn't exist.

### `/secure-dockerfile:fix`

Rewrite an existing Dockerfile to meet a target security tier. Runs an audit first, shows findings, then restructures the file.

```
/secure-dockerfile:fix                            # auto-find + interactive
/secure-dockerfile:fix path/to/Dockerfile         # specific file
/secure-dockerfile:fix --tier stig                # target STIG tier directly
```

Preserves your application's build logic while restructuring for security and size.

## Security Tiers

### Standard

Baseline every production Dockerfile should meet:

- Pinned base image tags
- Multi-stage builds
- Combined and cleaned RUN layers
- No secrets in the image
- HEALTHCHECK defined
- Proper layer ordering for cache efficiency

### Hardened

Defense-in-depth container security (includes Standard):

- Non-root USER
- Minimal runtime base (distroless, scratch, ubi-micro)
- No shell or package manager in runtime
- Capability dropping guidance
- Minimal package installation

### STIG

DoD STIG-level hardening (includes Hardened):

- Rootfs-builder pattern with `--installroot`
- STIG OS controls (core dumps, umask, PAM, GPG checking)
- FIPS-grade crypto configuration
- Scanner integration (trivy, grype)
- Build metadata labels
- Minimal langpack

## Output Formats

| Format | Flag | File | Use Case |
|--------|------|------|----------|
| Conversation | (default) | -- | Interactive review |
| Markdown | `--report` | `dockerfile-audit.md` | Documentation, PRs |
| SARIF | `--sarif` | `dockerfile-audit.sarif` | GitHub Code Scanning, CI/CD |

## License

MIT
