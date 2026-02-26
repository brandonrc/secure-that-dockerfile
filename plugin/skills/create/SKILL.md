---
name: create
description: Generate a new hardened Dockerfile from scratch by auto-detecting your project's language and framework. Supports security tiers (standard, hardened, stig).
argument-hint: [--tier standard|hardened|stig]
---

# Create Secure Dockerfile

Generate a production-ready, security-hardened Dockerfile tailored to the user's project. Follow every step below in order.

---

## Step 1: Parse Arguments

Check if the user passed `--tier` with a value of `standard`, `hardened`, or `stig`.

- If `--tier` was provided and the value is valid, store it and skip the tier selection step later.
- If `--tier` was provided with an invalid value, tell the user the valid options are `standard`, `hardened`, or `stig`, and ask them to choose.
- If `--tier` was not provided, you will ask the user to choose a tier after detection (Step 4).

---

## Step 2: Auto-Detect Project Language and Framework

Scan the user's project root for manifest files to determine the language and framework. Use the Glob tool to check for the following files:

| File(s) to check | Language/Framework |
|-------------------|--------------------|
| `Cargo.toml` | Rust |
| `package.json` | Node.js |
| `go.mod` | Go |
| `requirements.txt`, `Pipfile`, `pyproject.toml` | Python |
| `pom.xml`, `build.gradle`, `build.gradle.kts` | Java/Kotlin |
| `Gemfile` | Ruby |
| `mix.exs` | Elixir |
| `CMakeLists.txt`, `Makefile` | C/C++ |
| `*.csproj`, `*.fsproj` | .NET |

Run these glob checks in parallel to speed up detection.

**If multiple manifest types are found** (e.g., a monorepo with both `package.json` and `go.mod`), present the list to the user and ask which component they want to containerize.

**If no manifest files are found**, ask the user what language and framework the project uses.

**Additional detection:** While scanning, also look for these project-specific signals that will influence the Dockerfile:

- `frontend/` or `client/` directory -- indicates a frontend build step may be needed
- `.sqlx/` directory -- indicates SQLx offline mode for Rust (set `SQLX_OFFLINE=true`)
- `*.proto` or `proto/` directory -- indicates protobuf compilation is needed
- `tsconfig.json` -- indicates TypeScript compilation
- `next.config.*` or `nuxt.config.*` -- indicates a specific Node.js framework
- `Dockerfile` already exists -- note this for Step 8

---

## Step 3: Confirm Detection

Present the detection result to the user and ask for confirmation:

```
I detected this is a **[Language/Framework]** project based on `[manifest file]`.

Is this correct?
```

Wait for the user to confirm before proceeding. If they correct you, use their answer instead.

---

## Step 4: Select Security Tier

If `--tier` was not provided in Step 1, ask the user which security tier they want. Present the three options clearly:

- **Standard** -- Best practices baseline: multi-stage builds, pinned image tags, cache cleanup, HEALTHCHECK, minimal base images. Suitable for most production workloads.
- **Hardened** -- Everything in Standard plus: non-root user, minimal runtime image (distroless/scratch/ubi-micro), no shell or package manager in runtime, capability dropping guidance. For security-conscious deployments.
- **STIG** -- Everything in Hardened plus: rootfs-builder pattern, FIPS-validated crypto, STIG OS hardening controls (core dumps disabled, restrictive umask, PAM hardening), scanner integration stage, build metadata labels. For DoD/federal compliance or maximum security posture.

Wait for the user to choose before proceeding.

---

## Step 5: Load Knowledge Base

Read the following knowledge base files from the plugin directory to inform Dockerfile generation:

1. **`${CLAUDE_PLUGIN_ROOT}/knowledge/security-tiers.md`** -- Contains all security rules (SD001-SD010, HD001-HD008, ST001-ST011) with detailed descriptions, good/bad examples, and rationale. Use the rules for the selected tier and all tiers below it.

2. **`${CLAUDE_PLUGIN_ROOT}/knowledge/multi-stage.md`** -- Contains multi-stage build patterns organized by language, including dependency caching strategies (cargo-chef for Rust, package.json-first for Node, requirements.txt-first for Python), stage naming conventions, and the rootfs-builder pattern for STIG.

3. **`${CLAUDE_PLUGIN_ROOT}/knowledge/base-images.md`** -- Contains base image comparison data, tier compatibility matrix, size benchmarks, and a selection guide. Use this to pick the right builder and runtime base images for the detected language and selected tier.

Read all three files. They contain the authoritative rules and patterns that the generated Dockerfile must follow.

---

## Step 6: Generate the Dockerfile

Generate a complete, multi-stage Dockerfile that satisfies every rule for the selected tier (and all lower tiers). The Dockerfile must be tailored to the detected language and framework.

### Determine the application port

Before generating, check for port configuration in the project:

- Look for common config files (`config.toml`, `config.yaml`, `.env.example`, `application.properties`, `application.yml`, etc.)
- Check for port references in the main source files
- If the port cannot be determined, ask the user what port the application listens on

Do NOT assume a port. The EXPOSE and HEALTHCHECK instructions depend on the correct port.

### Stage structure by tier

**Standard tier** -- two stages:

```
builder -> runtime
```

**Hardened tier** -- three stages:

```
deps -> builder -> runtime
```

**STIG tier** -- seven stages:

```
base -> planner -> deps -> builder -> rootfs-builder -> scanners -> runtime
```

Not every language needs every stage. For example, Go does not benefit from a separate `planner` stage (that is a cargo-chef pattern for Rust). Adapt the stage count to the language while maintaining the tier's security requirements.

### Rules to satisfy per tier

**For ALL tiers (Standard baseline), the Dockerfile MUST:**

- Use pinned base image tags -- never `latest`, never bare tags (SD001)
- Use multi-stage builds separating build tools from runtime (SD002)
- Combine RUN layers with `&&` (SD003)
- Clean package manager caches in the same RUN layer as the install (SD004)
- Avoid embedding secrets in ENV, ARG, or COPY (SD006)
- Include a HEALTHCHECK instruction with `--interval`, `--timeout`, `--start-period`, and `--retries` (SD007)
- Use specific COPY paths instead of `COPY . .` wherever possible (SD008)
- Order layers for cache efficiency: dependencies before source code (SD009)
- Use a minimal base image appropriate for the language (SD010)

**Additionally for Hardened tier, the Dockerfile MUST also:**

- Create a non-root user and switch to it with USER before CMD/ENTRYPOINT (HD001)
- Use scratch, distroless, or ubi-micro as the runtime base (HD003)
- Use `--no-install-recommends` (apt) or `install_weak_deps=0` (dnf) for all package installs (HD004)
- Include comments documenting that the container should run with `--cap-drop=ALL` and listing any required capabilities (HD005)
- Ensure no package manager exists in the runtime image (HD006)
- Use `COPY --chown` instead of separate `RUN chown` commands (HD007)
- Include explicit EXPOSE for the application port(s) (HD008)

**Additionally for STIG tier, the Dockerfile MUST also:**

- Use the rootfs-builder pattern with `dnf --installroot` to construct the runtime filesystem (ST001)
- Disable core dumps: `* hard core 0` in `/etc/security/limits.d/` (ST002)
- Set restrictive umask: `UMASK 077` in `/etc/login.defs` (ST003)
- Remove `nullok` from all PAM configuration files (ST004)
- Set max concurrent login sessions: `* hard maxlogins 10` in `/etc/security/limits.d/` (ST005)
- Enable GPG checking: `localpkg_gpgcheck=1` in `dnf.conf` (ST006)
- Configure FIPS-grade crypto: set `fips=yes` in OpenSSL configuration (ST007)
- Clean machine-id: truncate `/etc/machine-id` to empty so each container gets a unique ID (ST008)
- Include a scanner integration stage with pinned Trivy and/or Grype (ST009)
- Add OCI-compliant build metadata LABELs using ARGs for `BUILD_DATE`, `VCS_REF`, `VERSION` (ST010)
- Use `glibc-minimal-langpack` instead of full locale packages (ST011)

### Language-specific requirements

Match the detected language's idiomatic build toolchain and dependency caching patterns. Reference the patterns from `multi-stage.md`:

| Language | Builder image | Build command | Dependency caching pattern | Runtime image (Standard) | Runtime image (Hardened/STIG) |
|----------|--------------|---------------|---------------------------|-------------------------|-------------------------------|
| **Rust** | `rust:<version>-slim` | `cargo build --release` | cargo-chef (planner + deps stages) | `debian:bookworm-slim` | `scratch` (musl) or `ubi-micro` (STIG) |
| **Go** | `golang:<version>` | `CGO_ENABLED=0 go build -ldflags="-s -w"` | `go.mod`/`go.sum` first, then `go mod download` | `alpine` | `distroless/static` or `scratch` |
| **Node.js** | `node:<version>-slim` | `npm ci && npm run build` | `package.json`/`package-lock.json` first | `node:<version>-slim` | `distroless/nodejs<version>-debian12` |
| **Python** | `python:<version>-slim` | `pip install --no-cache-dir --prefix=/install` | `requirements.txt` first | `python:<version>-slim` | `distroless/python3-debian12` |
| **Java** | `eclipse-temurin:<version>-jdk` | `./mvnw package -DskipTests` or `./gradlew build` | `pom.xml`/`build.gradle` first | `eclipse-temurin:<version>-jre-alpine` | `distroless/java<version>-debian12` |
| **Ruby** | `ruby:<version>-slim` | `bundle install` | `Gemfile`/`Gemfile.lock` first | `ruby:<version>-slim` | `ruby:<version>-alpine` (remove apk) |
| **Elixir** | `elixir:<version>-slim` | `mix release` | `mix.exs`/`mix.lock` first | `debian:bookworm-slim` | `debian:bookworm-slim` (stripped) |
| **.NET** | `mcr.microsoft.com/dotnet/sdk:<version>` | `dotnet publish -c Release` | `*.csproj` first, then `dotnet restore` | `mcr.microsoft.com/dotnet/aspnet:<version>` | `mcr.microsoft.com/dotnet/runtime-deps:<version>-alpine` |
| **C/C++** | `gcc:<version>` or `debian:bookworm-slim` | `cmake && make` or `make` | `CMakeLists.txt` first | `debian:bookworm-slim` | `distroless/cc` or `scratch` |

For STIG tier, all languages use UBI 9 as the builder base and `ubi-micro` as the runtime base with the rootfs-builder pattern. Refer to `${CLAUDE_PLUGIN_ROOT}/knowledge/examples/rust-ubi9.md` for the canonical STIG pattern.

### Dockerfile formatting and comments

- Start the Dockerfile with `# syntax=docker/dockerfile:1` to enable BuildKit features
- Add a header comment block explaining what the Dockerfile does, the security tier, and which rules it satisfies
- Add a comment before each stage explaining its purpose
- Add inline comments for any non-obvious instruction (e.g., why `CGO_ENABLED=0`, why `--no-install-recommends`, why a specific file is copied)
- Use exec-form (JSON array) for CMD and ENTRYPOINT, never shell-form
- Use lowercase with hyphens for stage names (`rootfs-builder`, not `RootfsBuilder`)
- Reference stages by name in `COPY --from=`, never by numeric index

---

## Step 7: Check for .dockerignore

Check if a `.dockerignore` file already exists in the project root.

**If `.dockerignore` does NOT exist**, generate one appropriate for the detected language. Include at minimum:

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

Then add language-specific patterns:

- **Node.js:** `node_modules`, `npm-debug.log*`, `coverage`, `.nyc_output`, `dist`
- **Python:** `__pycache__`, `*.pyc`, `*.pyo`, `.venv`, `venv`, `.pytest_cache`, `.mypy_cache`, `*.egg-info`
- **Rust:** `target`, `*.rs.bk`
- **Go:** `vendor` (if using modules), `*.test`
- **Java/Kotlin:** `target`, `build`, `*.class`, `*.jar`, `.gradle`
- **Ruby:** `vendor/bundle`, `.bundle`, `coverage`, `tmp`
- **Elixir:** `_build`, `deps`, `.elixir_ls`
- **.NET:** `bin`, `obj`, `*.user`, `*.suo`
- **C/C++:** `build`, `*.o`, `*.a`, `*.so`, `*.out`

Show the user the `.dockerignore` content and confirm before writing.

**If `.dockerignore` already exists**, read it and check if it covers the essential patterns. If it is missing critical entries (like `.git` or `.env`), suggest additions but do not overwrite without asking.

---

## Step 8: Write Files

Before writing, check if a `Dockerfile` already exists in the project root.

**If a Dockerfile already exists:**

- Inform the user that a Dockerfile already exists
- Show a brief diff summary of what would change
- Ask if they want to overwrite it, write to a different filename (e.g., `Dockerfile.secure`), or cancel
- Wait for their answer before proceeding

**If no Dockerfile exists (or user confirmed overwrite):**

- Write the Dockerfile to the project root
- Write `.dockerignore` if one was generated in Step 7

**After writing, provide a summary that includes:**

1. A brief explanation of each stage in the generated Dockerfile and its purpose
2. The security tier applied and which rule IDs are satisfied
3. A sample build command:
   - Standard/Hardened: `docker build -t <app-name>:latest .`
   - STIG: `docker build --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) --build-arg VCS_REF=$(git rev-parse HEAD) --build-arg VERSION=0.1.0 -t <app-name>:latest .`
4. A sample run command with security flags appropriate to the tier:
   - Standard: `docker run -p <port>:<port> <app-name>:latest`
   - Hardened: `docker run --cap-drop=ALL -p <port>:<port> --read-only <app-name>:latest`
   - STIG: `docker run --cap-drop=ALL --security-opt=no-new-privileges -p <port>:<port> --read-only --tmpfs /tmp <app-name>:latest`
5. If scanner stages exist (STIG tier), explain how to run the scanner stage:
   `docker build --target scanners .`

---

## Important Guidelines

- **Match the language's toolchain exactly.** Use `cargo` for Rust, `go build` for Go, `npm ci` for Node.js, `pip install` for Python, `mvn` or `gradle` for Java, `bundle install` for Ruby, `mix release` for Elixir, `dotnet publish` for .NET.
- **Use dependency caching patterns from multi-stage.md.** For example, use cargo-chef for Rust, copy `package.json` first for Node.js, copy `requirements.txt` first for Python. These patterns dramatically improve build times.
- **Pick base images from base-images.md based on the tier.** The tier compatibility matrix in that file defines which base images are acceptable for each tier.
- **Account for project-specific needs.** If you detect a `frontend/` directory, add a frontend build stage. If you detect `.sqlx/`, set `SQLX_OFFLINE=true`. If you detect protobuf files, add protoc installation to the builder.
- **Do not assume ports.** Check config files or ask the user. Wrong port numbers make the HEALTHCHECK and EXPOSE incorrect.
- **Do not assume the binary name.** Read `Cargo.toml`, `package.json`, `go.mod`, or equivalent to determine the correct binary or entrypoint name.
- **Always use exec-form for CMD and ENTRYPOINT.** Shell-form prevents proper signal handling and breaks graceful shutdown.
- **For STIG tier, always reference the canonical example** at `${CLAUDE_PLUGIN_ROOT}/knowledge/examples/rust-ubi9.md` for the rootfs-builder and STIG hardening patterns. Adapt the language-specific parts but keep the hardening controls identical.
