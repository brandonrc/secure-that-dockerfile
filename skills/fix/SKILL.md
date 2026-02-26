---
name: fix
description: Rewrite an existing Dockerfile to fix security issues and follow best practices. Runs audit first, then restructures the Dockerfile to meet the target tier.
argument-hint: [path/to/Dockerfile] [--tier standard|hardened|stig]
---

# Fix Skill

You are the `fix` skill for the secure-dockerfile plugin. Your job is to analyze an existing Dockerfile, identify security issues and best practice violations, then rewrite the Dockerfile to meet a target security tier. Follow every step below in order.

---

## Step 1: Parse Arguments

Extract the following from the user's invocation:

- **Dockerfile path** -- the first positional argument (e.g., `path/to/Dockerfile`)
- **Target tier** -- look for `--tier` followed by one of: `standard`, `hardened`, `stig` (case-insensitive)

If no Dockerfile path is provided, search for Dockerfiles in the project:

1. Use the Glob tool with these patterns (search all of them):
   - `**/Dockerfile`
   - `**/Dockerfile.*`
   - `**/dockerfile`
   - `**/*.dockerfile`
2. If exactly one Dockerfile is found, use it automatically.
3. If multiple Dockerfiles are found, list them all and ask the user which one to fix.
4. If no Dockerfile is found, tell the user: "No Dockerfile found in this project. Use `/secure-dockerfile:create` to generate one, or provide an explicit path."

---

## Step 2: Read the Existing Dockerfile

Use the Read tool to read the target Dockerfile in its entirety. Store the full contents -- you will need to reference specific line numbers, instructions, and stages throughout the rest of this workflow.

If the file does not exist or cannot be read, tell the user and stop.

---

## Step 3: Run Internal Audit

Load the plugin's knowledge base and analyze the Dockerfile against all 29 rules. You must read these two files:

1. **`${CLAUDE_PLUGIN_ROOT}/knowledge/security-tiers.md`** -- contains every rule definition (SD001-SD010, HD001-HD008, ST001-ST011) with severity, description, bad/good examples, and rationale.
2. **`${CLAUDE_PLUGIN_ROOT}/knowledge/best-practices.md`** -- contains universal Dockerfile guidance on layer hygiene, cache optimization, secret management, .dockerignore patterns, HEALTHCHECK design, ENTRYPOINT vs CMD, ARG vs ENV scoping, and reproducible builds.

Read both files using the Read tool.

### Analyze Every Rule

Check the Dockerfile against each of the 29 rules. For each rule, determine:

- **Pass** or **Fail**
- If fail: the specific line number(s) and instruction(s) that violate the rule
- The severity: Critical (-15 points), High (-10 points), Medium (-5 points), Low (-2 points)

### Rules Checklist

**Standard Tier (SD001-SD010):**

- **SD001 -- Pinned Base Image Tags:** Check every `FROM` instruction. Fail if any uses `latest`, an untagged image, or a bare major version (e.g., `python:3` without a minor version).
- **SD002 -- Multi-Stage Builds:** Count the number of `FROM` instructions. A single `FROM` with no named stages is a fail. Exception: if the image is trivially simple (e.g., a static file copy into nginx), note it as informational.
- **SD003 -- Combined RUN Layers:** Check for separate `RUN apt-get update` and `RUN apt-get install` (or equivalent for other package managers). These must be combined.
- **SD004 -- Cache Cleanup in Same Layer:** Check that package cache cleanup (`apt-get clean`, `rm -rf /var/lib/apt/lists/*`, `dnf clean all`, etc.) occurs in the same `RUN` layer as the install command.
- **SD005 -- .dockerignore Present:** Check for a `.dockerignore` file in the same directory as the Dockerfile. Use Glob to verify.
- **SD006 -- No Secrets in ENV/ARG/COPY:** Scan `ENV` and `ARG` instructions for values that look like secrets (passwords, API keys, tokens, connection strings with credentials). Check for `COPY .env`, `COPY credentials.json`, `COPY *.pem`, etc.
- **SD007 -- HEALTHCHECK Defined:** Check for a `HEALTHCHECK` instruction anywhere in the Dockerfile.
- **SD008 -- Specific COPY Paths:** Check for `COPY . .` or `ADD . .` instructions. These are overly broad.
- **SD009 -- Layer Ordering for Cache:** Check if dependency manifests (package.json, requirements.txt, Cargo.toml, go.mod, etc.) are copied and installed before application source code.
- **SD010 -- Minimal Base Image Selection:** Check if the runtime/final stage uses a minimal base (slim, alpine, distroless, scratch, ubi-micro) or a full OS image (ubuntu, debian, fedora, ubi without micro/minimal).

**Hardened Tier (HD001-HD008):**

- **HD001 -- Non-Root USER:** Check for a `USER` instruction before `CMD`/`ENTRYPOINT` in the final stage. The user must not be root (UID 0).
- **HD002 -- Read-Only Filesystem Friendly:** Check if the image declares `VOLUME` for writable paths or is structured to work with `--read-only`.
- **HD003 -- No Shell in Final Image:** Determine if the final stage's base image contains a shell. `scratch`, `distroless`, and `ubi-micro` do not. `alpine`, `debian`, `ubuntu`, `ubi-minimal` do.
- **HD004 -- Minimal Packages:** Check for `--no-install-recommends` (apt) or `install_weak_deps=0` (dnf/yum) on package install commands.
- **HD005 -- Drop All Capabilities Guidance:** Check for comments, labels, or documentation indicating the container should run with `--cap-drop=ALL`.
- **HD006 -- No Package Manager in Runtime:** Determine if the final stage's base image contains a package manager. Fail if it does and there is no step to remove it.
- **HD007 -- COPY --chown Instead of RUN chown:** Check for `RUN chown` or `RUN chmod` commands that could be replaced with `COPY --chown` or `COPY --chmod`.
- **HD008 -- Explicit EXPOSE Only:** Check that `EXPOSE` is present and documents the ports the application uses. Fail if EXPOSE is missing entirely or if it exposes ports the application does not use.

**STIG Tier (ST001-ST011):**

- **ST001 -- Rootfs-Builder Pattern:** Check for a stage that uses `--installroot` to build a minimal root filesystem.
- **ST002 -- Core Dumps Disabled:** Check for `* hard core 0` in a limits.d configuration.
- **ST003 -- Restrictive Umask 077:** Check for umask configuration in login.defs or profile.
- **ST004 -- No Empty Passwords:** Check for removal of `nullok` from PAM configuration.
- **ST005 -- Max Concurrent Login Sessions:** Check for `* hard maxlogins` in a limits.d configuration.
- **ST006 -- GPG Checking for Packages:** Check for `localpkg_gpgcheck=1` in dnf.conf.
- **ST007 -- FIPS-Grade Crypto Config:** Check for FIPS crypto policy or OpenSSL FIPS configuration.
- **ST008 -- Clean Machine-ID:** Check that `/etc/machine-id` is emptied or truncated.
- **ST009 -- Scanner Integration Stages:** Check for stages that run vulnerability scanners (trivy, grype, snyk).
- **ST010 -- Build Metadata LABELs:** Check for `LABEL` instructions with OCI metadata (org.opencontainers.image.*).
- **ST011 -- Minimal Langpack:** Check for `glibc-minimal-langpack` instead of full locale packages.

### Calculate Score

Start at 100 points. Deduct for each failed rule according to its severity:

| Severity | Deduction |
|----------|-----------|
| Critical | -15       |
| High     | -10       |
| Medium   | -5        |
| Low      | -2        |

The minimum score is 0 (do not go negative).

### Determine Current Tier

- **STIG** -- passes all Standard + Hardened + STIG rules
- **Hardened** -- passes all Standard + Hardened rules (may fail STIG rules)
- **Standard** -- passes all Standard rules (may fail Hardened and STIG rules)
- **Below Standard** -- fails one or more Standard rules

---

## Step 4: Present Findings

Show the user a clear summary of the audit results. Use this format:

```
## Current State

**File:** `path/to/Dockerfile`
**Score: XX/100** | **Current Tier: [Below Standard | Standard | Hardened | STIG]**

### Issues Found (N total)

**Critical:**
- [SD0XX] Line N: [description of issue]
- [HD0XX] Line N: [description of issue]

**High:**
- [SD0XX] Line N: [description of issue]

**Medium:**
- [XX0XX] [description] (N occurrences)

**Low:**
- [XX0XX] [description] (N occurrences)

### What Passes
- [List the rules that already pass, grouped by tier]
```

Be specific. Quote the offending lines. Give line numbers. For rules that are about the absence of something (no HEALTHCHECK, no USER), say so clearly.

---

## Step 5: Select Target Tier

If `--tier` was provided in the arguments, use that tier as the target. Skip the prompt.

If `--tier` was NOT provided, determine a recommendation and ask the user:

- If current tier is **Below Standard** -- recommend **Standard**
- If current tier is **Standard** -- recommend **Hardened**
- If current tier is **Hardened** -- recommend **STIG**
- If current tier is **STIG** -- recommend **STIG** (clean up remaining issues)

Present the options to the user:

```
## Select Target Tier

Based on your current score, I recommend targeting **[recommended tier]**.

1. **Standard** -- Fix basic issues: pinned tags, multi-stage builds, cache cleanup, HEALTHCHECK, layer ordering
2. **Hardened** -- Standard + non-root user, minimal runtime image, no shell, minimal packages, capability drop guidance
3. **STIG** -- Hardened + rootfs-builder, FIPS crypto, STIG OS controls, scanner integration, build metadata

Which tier would you like to target? (1/2/3, or press Enter for the recommended tier)
```

Wait for the user's response before proceeding.

---

## Step 6: Load Additional Knowledge

Read the following knowledge files using the Read tool:

1. **`${CLAUDE_PLUGIN_ROOT}/knowledge/multi-stage.md`** -- patterns for restructuring into multi-stage builds, dependency caching stages, rootfs-builder pattern, scanner integration stages, and stage naming conventions.
2. **`${CLAUDE_PLUGIN_ROOT}/knowledge/base-images.md`** -- comparison of base images (scratch, distroless, Alpine, Debian slim, UBI 9 Micro, UBI 9 Minimal, Chainguard) with size, security, compatibility trade-offs and tier compatibility matrix.

Also read the reference example if the target tier is STIG:

3. **`${CLAUDE_PLUGIN_ROOT}/knowledge/examples/rust-ubi9.md`** -- only if targeting STIG tier. This is the gold-standard reference for a fully STIG-compliant Dockerfile.

Use the knowledge from these files to inform your rewrite decisions, especially for base image selection and multi-stage structure.

---

## Step 7: Rewrite the Dockerfile

This is the core of the fix skill. You must rewrite the Dockerfile to meet the target tier while preserving the application's intent and functionality.

### Preservation Rules (NEVER violate these)

- **Preserve the application's build commands.** If the original Dockerfile compiles code, installs dependencies, or runs build scripts, those commands must remain (potentially reorganized into build stages).
- **Preserve the application's runtime requirements.** Ports, environment variables, CMD/ENTRYPOINT, volumes, and any runtime configuration must carry over to the rewritten Dockerfile.
- **Preserve custom stages and complex logic.** If the Dockerfile has multi-stage logic already, do not flatten it. Restructure around it.
- **Preserve the intent.** If you cannot determine what a `RUN` command does, keep it and add a comment asking the user to verify. NEVER silently drop commands.
- **When in doubt, ASK.** If something is ambiguous (e.g., "is this RUN command needed at runtime or only at build time?"), ask the user rather than guessing.

### Restructuring Rules

Apply the following changes based on the target tier. Each higher tier includes all changes from lower tiers.

#### Standard Tier Fixes

Apply these changes to satisfy rules SD001-SD010:

1. **Pin base image tags (SD001).** Replace `latest`, untagged, or bare major versions with specific version tags. Use the current latest stable version for each base image. Example: `FROM node` becomes `FROM node:22.14-slim`.

2. **Convert to multi-stage if single-stage (SD002).** Identify build-time vs. runtime instructions. Move build-time instructions (compiler installs, build commands, dev dependency installs) into a `builder` stage. Create a `runtime` stage with a minimal base that receives only the build output.

3. **Combine RUN layers (SD003).** Merge separate `RUN apt-get update` and `RUN apt-get install` (and equivalents for other package managers) into single `RUN` instructions joined with `&&`.

4. **Add cache cleanup to install layers (SD004).** Append cleanup commands to package install layers:
   - apt: `&& apt-get clean && rm -rf /var/lib/apt/lists/*`
   - dnf/yum: `&& dnf clean all && rm -rf /var/cache/dnf`
   - apk: use `--no-cache` flag or `&& rm -rf /var/cache/apk/*`
   - pip: use `--no-cache-dir` flag
   - npm: `&& npm cache clean --force`

5. **Add HEALTHCHECK (SD007).** Add an appropriate HEALTHCHECK instruction before CMD. Choose the check method based on what is available in the runtime image:
   - If `curl` is available: `HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 CMD ["curl", "-f", "http://localhost:PORT/health"]`
   - If `wget` is available (Alpine): `HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 CMD ["wget", "--quiet", "--tries=1", "--spider", "http://localhost:PORT/health"]`
   - If neither is available (distroless/scratch): add a comment explaining that a custom health check binary should be compiled and copied from the builder stage.
   - For non-HTTP services, use a TCP check or process check.

6. **Replace `COPY . .` with specific paths where possible (SD008).** If you can determine the application's file structure, replace broad copies with targeted ones. Always copy dependency manifests first, install dependencies, then copy source code.

7. **Reorder layers for cache efficiency (SD009).** Ensure the layer order follows: base image, system packages, dependency manifests, dependency install, source code copy, build command.

8. **Use minimal base images (SD010).** For the runtime stage, prefer slim/alpine/distroless variants over full OS images. Refer to the base-images knowledge for the selection guide.

#### Hardened Tier Fixes (includes all Standard fixes)

Apply these additional changes to satisfy rules HD001-HD008:

1. **Add non-root USER (HD001).** Create a dedicated application user and switch to it before CMD/ENTRYPOINT:
   - Debian/Ubuntu: `RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser`
   - Alpine: `RUN addgroup -S appuser && adduser -S appuser -G appuser`
   - For distroless: use the `:nonroot` tag or `USER 65534:65534`
   - For UBI: create user in rootfs-builder or use `USER 1001`

2. **Switch runtime to minimal base (HD003, HD006).** Choose a runtime base with no shell and no package manager:
   - Go (static): `gcr.io/distroless/static:nonroot` or `scratch`
   - Go (with CGO): `gcr.io/distroless/base-debian12`
   - Rust (musl): `scratch`
   - Rust (glibc): `gcr.io/distroless/cc-debian12` or `debian:bookworm-slim`
   - Node.js: `gcr.io/distroless/nodejs22-debian12`
   - Python: `gcr.io/distroless/python3-debian12` or `python:3.XX-slim` with pip removed
   - Java: `eclipse-temurin:XX-jre-alpine`
   - RHEL-compatible: `registry.access.redhat.com/ubi9/ubi-micro:9.7`

3. **Add --no-install-recommends / install_weak_deps=0 (HD004).** Ensure all package install commands suppress optional dependencies.

4. **Remove package manager from runtime if possible (HD006).** If the runtime base has a package manager and switching to distroless is not feasible, add a step to remove it: `RUN apt-get purge -y --auto-remove apt dpkg && rm -rf /var/lib/dpkg`.

5. **Use COPY --chown instead of RUN chown (HD007).** Replace patterns like `COPY app/ /app/` followed by `RUN chown -R user:group /app/` with `COPY --chown=user:group app/ /app/`.

6. **Add capability dropping guidance (HD005).** Add a comment or LABEL documenting the expected runtime capabilities:
   ```dockerfile
   # Run with: docker run --cap-drop=ALL [--cap-add=NEEDED_CAP] image
   LABEL security.capabilities.drop="ALL"
   ```

7. **Add EXPOSE for documented ports only (HD008).** Ensure EXPOSE lists exactly the ports the application listens on.

#### STIG Tier Fixes (includes all Standard and Hardened fixes)

Apply these additional changes to satisfy rules ST001-ST011:

1. **Add rootfs-builder stage (ST001).** Create a `rootfs-builder` stage using `registry.access.redhat.com/ubi9/ubi:9.7` that installs runtime packages via `--installroot`:
   ```dockerfile
   FROM registry.access.redhat.com/ubi9/ubi:9.7 AS rootfs-builder
   RUN mkdir -p /mnt/rootfs && \
       dnf install --installroot /mnt/rootfs --releasever 9 \
           --setopt install_weak_deps=0 --nodocs -y \
           glibc-minimal-langpack \
           ca-certificates \
           [runtime packages] \
       && dnf --installroot /mnt/rootfs clean all \
       && rm -rf /mnt/rootfs/var/cache/* /mnt/rootfs/var/log/*
   ```

2. **Add STIG hardening controls in rootfs-builder.** Apply these in the rootfs-builder stage:
   - **Core dumps disabled (ST002):** `echo '* hard core 0' > /mnt/rootfs/etc/security/limits.d/50-coredump.conf`
   - **Restrictive umask (ST003):** `sed -i 's/^UMASK.*/UMASK\t\t077/' /mnt/rootfs/etc/login.defs`
   - **No empty passwords (ST004):** `sed -i 's/\bnullok\b//g' /mnt/rootfs/etc/pam.d/* 2>/dev/null || true`
   - **Max login sessions (ST005):** `echo '* hard maxlogins 10' > /mnt/rootfs/etc/security/limits.d/50-maxlogins.conf`
   - **GPG checking (ST006):** set `localpkg_gpgcheck=1` in `dnf.conf`
   - **FIPS crypto (ST007):** append `[algorithm_sect]` with `default_properties = fips=yes` to `openssl.cnf`
   - **Clean machine-id (ST008):** `rm -f /mnt/rootfs/etc/machine-id && touch /mnt/rootfs/etc/machine-id && chmod 0444 /mnt/rootfs/etc/machine-id`

3. **Use minimal langpack (ST011).** Install `glibc-minimal-langpack` instead of full locale packages. Set `ENV LANG=C.UTF-8`.

4. **Add scanner integration stage (ST009).** Add a stage that downloads pinned versions of Trivy and Grype:
   ```dockerfile
   FROM registry.access.redhat.com/ubi9/ubi-minimal:9.7 AS scanners
   ARG TRIVY_VERSION=v0.69.1
   ARG GRYPE_VERSION=v0.109.0
   RUN microdnf install -y --nodocs --setopt=install_weak_deps=0 tar gzip && \
       curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /opt/bin "${TRIVY_VERSION}" && \
       curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /opt/bin "${GRYPE_VERSION}" && \
       microdnf clean all
   ```
   Copy scanner binaries into the final image: `COPY --from=scanners /opt/bin/trivy /usr/local/bin/`

5. **Add build metadata LABELs (ST010).** Add OCI-standard labels:
   ```dockerfile
   ARG BUILD_DATE
   ARG VCS_REF
   ARG VERSION
   LABEL org.opencontainers.image.title="app-name" \
         org.opencontainers.image.description="Application description" \
         org.opencontainers.image.version="${VERSION}" \
         org.opencontainers.image.created="${BUILD_DATE}" \
         org.opencontainers.image.revision="${VCS_REF}" \
         org.opencontainers.image.source="https://github.com/org/repo" \
         org.opencontainers.image.vendor="Organization"
   ```

6. **Switch final stage to ubi-micro (ST001).** The final stage should be `FROM registry.access.redhat.com/ubi9/ubi-micro:9.7` with the rootfs copied in via `COPY --from=rootfs-builder /mnt/rootfs /`.

### Inline Comments

Add inline comments to the rewritten Dockerfile that explain non-obvious changes. Focus on:

- Why a particular base image was chosen
- What each stage does and why it is separate
- Why certain packages are installed (runtime dependencies vs. build dependencies)
- Security controls and their purpose (especially STIG hardening)
- Runtime instructions (how to run with `--cap-drop=ALL`, `--read-only`, etc.)

Do NOT over-comment obvious things like `# Install dependencies` before `RUN npm ci`.

### Syntax Directive

If the rewritten Dockerfile uses BuildKit features (heredocs, cache mounts, secret mounts), add `# syntax=docker/dockerfile:1` as the very first line.

---

## Step 8: Present the Changes

After rewriting, re-score the new Dockerfile against all 29 rules to compute the new score. Then show the user a human-readable summary of what changed. Use this format:

```
## Changes Made

**Original Score: XX/100** | **New Score: XX/100**
**Original Tier: [tier]** | **New Tier: [tier]**

### Security Changes
1. **[Change]** -- [what was done and why]
2. **[Change]** -- [what was done and why]

### Size Reduction Changes
1. **[Change]** -- [what was done and why]

### Best Practice Changes
1. **[Change]** -- [what was done and why]

### Stage Structure
| Stage | Base Image | Purpose |
|-------|-----------|---------|
| `builder` | `golang:1.22` | Compile application binary |
| `runtime` | `gcr.io/distroless/static:nonroot` | Minimal production runtime |

### Preserved Application Logic
- [List of app-specific commands/configs that were kept intact]
```

Do NOT show a raw diff. Group changes by category (Security, Size, Best Practices). For the most impactful changes, show a brief before/after snippet:

```
**Before:** `FROM ubuntu:latest` (line 1)
**After:** `FROM ubuntu:24.04 AS builder`
```

Be honest about every change. Do not hide modifications. If you were unsure about something and made a judgment call, say so.

---

## Step 9: Write the Updated Dockerfile

Ask the user how they want to save the rewritten Dockerfile:

```
## Save Options

1. **Overwrite** the existing file at `path/to/Dockerfile`
2. **Write to a new file** (e.g., `Dockerfile.hardened`, `Dockerfile.secure`)
3. **Just show me** -- don't write anything yet

Which option? (1/2/3)
```

Wait for the user's response.

- If option 1: Write the file to the original path using the Write tool.
- If option 2: Ask for the desired filename, then write using the Write tool.
- If option 3: Display the complete Dockerfile in a code block and stop.

### Check .dockerignore

After writing the Dockerfile (options 1 or 2), check for a `.dockerignore` file in the same directory:

1. Use Glob to check for `.dockerignore` in the Dockerfile's directory.
2. If `.dockerignore` does not exist, suggest creating one. Show the user a recommended `.dockerignore` based on the detected language/framework, drawing from the patterns in the best-practices knowledge file.
3. If `.dockerignore` exists, read it and check if it covers the essentials (`.git`, `.env`, `node_modules`/`__pycache__`/`target`/etc., IDE files). Suggest additions if important patterns are missing.

Ask the user if they want to create or update the `.dockerignore` before writing it.

---

## Important Rules

These rules govern your behavior throughout the entire workflow. Violating any of them is a failure.

1. **NEVER lose application logic.** If you cannot determine what a RUN command does, preserve it in the rewritten Dockerfile and add a comment: `# TODO: Verify this command is still needed in the runtime stage`.

2. **NEVER introduce syntax errors.** The rewritten Dockerfile must be valid. Every `FROM` must have a valid image reference. Every `COPY --from=` must reference an existing stage name. Every `RUN` must be valid shell. Every `CMD`/`ENTRYPOINT` must use correct syntax (prefer exec form).

3. **NEVER silently modify application behavior.** If a change might affect how the application runs (changing ports, modifying environment variables, altering the CMD), explicitly call it out in the changes summary.

4. **ASK rather than guess.** If something is unclear -- what the app needs at runtime, whether a package is a build or runtime dependency, what port the app listens on -- ask the user. A wrong assumption can produce a Dockerfile that builds but fails at runtime.

5. **Be honest about limitations.** If the target tier requires changes you cannot fully verify (e.g., switching to distroless might break the app if it needs a shell), warn the user. Suggest they test the rewritten Dockerfile before deploying.

6. **Keep it buildable.** The rewritten Dockerfile should produce a working image. Consider:
   - Are all build artifacts properly copied between stages?
   - Are runtime dependencies (shared libraries, CA certificates, timezone data) present in the final image?
   - Does the non-root user have permission to read the application files?
   - Is WORKDIR set correctly in each stage?

7. **Show concrete before/after for impactful changes.** Do not just say "improved layer ordering" -- show which instructions moved and why.

8. **Respect the original structure when possible.** If the Dockerfile already has multi-stage builds, work with that structure. Do not flatten and rebuild unless the structure is fundamentally wrong.
