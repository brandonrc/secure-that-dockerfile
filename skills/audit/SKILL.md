---
name: audit
description: Analyze a Dockerfile for security issues, size bloat, and best practice violations. Supports --report (markdown) and --sarif output formats.
argument-hint: [path/to/Dockerfile] [--report] [--sarif]
---

# Audit Skill

You are performing a comprehensive security and best-practices audit of a Dockerfile. Follow each step below in order.

## Step 1: Parse Arguments

Extract the following from the user's invocation:

- **Path argument:** The first non-flag argument is the path to a Dockerfile (e.g., `/secure-dockerfile:audit path/to/Dockerfile`).
- **Flags:** Check for `--report` and `--sarif` anywhere in the arguments.

If no path argument was provided, search for Dockerfiles in the project:

1. Use Glob to search for `**/Dockerfile*`, `**/dockerfile*`, and `**/*.dockerfile`.
2. If exactly one Dockerfile is found, use it.
3. If multiple Dockerfiles are found, list them and ask the user which one to audit. Do not proceed until the user chooses.
4. If no Dockerfile is found, tell the user and stop.

## Step 2: Read the Dockerfile

Use the Read tool to read the target Dockerfile in full. Record the complete content with line numbers -- you will reference specific line numbers in your findings.

## Step 3: Load Knowledge Base

Read both knowledge base files using the Read tool:

1. `${CLAUDE_PLUGIN_ROOT}/knowledge/security-tiers.md` -- contains all 29 rules (SD001-SD010, HD001-HD008, ST001-ST011) with descriptions, severity levels, and good/bad examples.
2. `${CLAUDE_PLUGIN_ROOT}/knowledge/best-practices.md` -- contains universal Dockerfile guidance on layer hygiene, caching, secrets, .dockerignore, HEALTHCHECK, ENTRYPOINT/CMD, and ARG/ENV scoping.

These files are the authoritative source for what constitutes a violation. Use the exact rule IDs and severity levels defined there.

## Step 4: Analyze the Dockerfile

Check every rule against the Dockerfile content. For each rule, determine whether the Dockerfile violates it, passes it, or whether the rule is not applicable. Reference specific line numbers for every finding.

### Standard Tier (SD prefix)

**SD001 -- Pinned Base Image Tags (Critical)**
Inspect every `FROM` instruction. Flag any that use:
- The `:latest` tag explicitly
- No tag at all (e.g., `FROM ubuntu`)
- A bare major version only (e.g., `FROM python:3`)
- Acceptable: specific minor/patch tags (`python:3.12-slim`), digest pins (`@sha256:...`)

**SD002 -- Multi-Stage Builds (High)**
Count the number of `FROM` instructions. If there is only one `FROM`, flag this as a violation. Multi-stage builds separate build-time dependencies from the runtime image.

**SD003 -- Combined RUN Layers (Medium)**
Look for patterns where `apt-get update`, `apt-get install`, `dnf install`, `yum install`, `apk add`, or similar package operations are split across separate `RUN` instructions. They should be combined with `&&` in a single `RUN`.

**SD004 -- Cache Cleanup in Same Layer (Medium)**
For every `RUN` instruction that installs packages, check that the same `RUN` instruction also cleans up caches. Look for:
- apt: `apt-get clean && rm -rf /var/lib/apt/lists/*`
- dnf/yum: `dnf clean all && rm -rf /var/cache/dnf`
- apk: `rm -rf /var/cache/apk/*` or use of `--no-cache` flag
- pip: `--no-cache-dir` flag
- npm: `npm cache clean --force`
A cleanup command in a separate `RUN` layer does not count -- it must be in the same layer.

**SD005 -- .dockerignore Present (Medium)**
Check the filesystem for a `.dockerignore` file in the same directory as the Dockerfile. Use Glob to search for `.dockerignore` in that directory. This check is about the filesystem, not the Dockerfile content.

**SD006 -- No Secrets in ENV/ARG/COPY (Critical)**
Scan all `ENV`, `ARG`, `COPY`, and `ADD` instructions for indicators of secrets:
- Variable names containing: `PASSWORD`, `SECRET`, `KEY`, `TOKEN`, `CREDENTIAL`, `API_KEY`, `PRIVATE`, `AUTH`
- Values that look like credentials: connection strings with passwords, AWS keys (`AKIA...`), GitHub tokens (`ghp_...`, `gho_...`), generic high-entropy strings
- Files being copied: `.env`, `credentials.json`, `*.pem`, `*.key`, `.npmrc` (when not using BuildKit secrets), `.netrc`
- `RUN` instructions that write secrets to files (e.g., `echo "password=..." > config`)

**SD007 -- HEALTHCHECK Defined (Medium)**
Search the Dockerfile for a `HEALTHCHECK` instruction. If none is present, flag it. Note: `HEALTHCHECK NONE` explicitly disables health checking and should also be flagged.

**SD008 -- Specific COPY Paths (Low)**
Look for `COPY . .` or `COPY . /` or `ADD . .` instructions that copy the entire build context. Specific file and directory copies are preferred.

**SD009 -- Layer Ordering for Cache (Low)**
Check whether dependency manifest files (e.g., `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `Gemfile`) are copied and their install commands run before the bulk source code copy. If source code is copied first (via `COPY . .`) and then dependencies are installed, flag it.

**SD010 -- Minimal Base Image Selection (Medium)**
Check the final stage's `FROM` image. Flag if it uses a full OS image when a minimal variant exists. Minimal indicators include: `-slim`, `-alpine`, `distroless`, `ubi-minimal`, `ubi-micro`, `scratch`, `-bookworm-slim`, `-bullseye-slim`. Full images like `ubuntu:24.04`, `debian:bookworm`, `node:20`, `python:3.12` (without a slim/alpine suffix) should be flagged.

### Hardened Tier (HD prefix)

**HD001 -- Non-Root USER (Critical)**
Search for a `USER` instruction in the final build stage that switches to a non-root user. `USER root` does not count. The `USER` instruction must appear before or without being overridden back to root before `CMD`/`ENTRYPOINT`. Also check for images that inherently run as non-root (e.g., `distroless/static:nonroot`).

**HD002 -- Read-Only Filesystem Friendly (Medium)**
Check if the image writes to paths at runtime that would fail with `--read-only`. Look for:
- `RUN mkdir` for runtime data directories without corresponding `VOLUME` declarations
- Absence of `VOLUME` declarations for known writable paths (logs, cache, temp, data directories)
- Write operations in `CMD`/`ENTRYPOINT` scripts
This is a best-effort heuristic. Flag if there is no evidence the image is designed for read-only operation (no `VOLUME` instructions and the application likely writes to disk).

**HD003 -- No Shell in Final Image (Medium)**
Check if the final stage base image is shell-free. Shell-free images include: `scratch`, images in the `gcr.io/distroless/` family, `ubi-micro`. Images like `debian`, `ubuntu`, `alpine`, `-slim` variants, and `ubi-minimal` all include a shell. Also check if the Dockerfile uses shell form for `CMD`/`ENTRYPOINT` (which requires a shell).

**HD004 -- Minimal Packages (Medium)**
For every `apt-get install` invocation, check for the `--no-install-recommends` flag. For `dnf` or `yum`, check for `--setopt=install_weak_deps=0` or that `install_weak_deps=0` is set in `dnf.conf`. Missing these flags means extra packages are being installed.

**HD005 -- Drop All Capabilities Guidance (High)**
Look for comments, labels, or documentation within the Dockerfile that instruct operators to run with `--cap-drop=ALL`. Check for:
- Comments containing `cap-drop` or `capabilities`
- Labels like `security.capabilities.drop`
- A `docker run` example in comments that includes `--cap-drop=ALL`

**HD006 -- No Package Manager in Runtime (Medium)**
Determine if the final runtime image still has a package manager available. If the final stage uses a base that includes apt, dnf, yum, apk, pip, or npm, and the Dockerfile does not explicitly remove them, flag this. Images based on `scratch`, `distroless`, or `ubi-micro` pass automatically.

**HD007 -- COPY --chown Instead of RUN chown (Low)**
Search for `RUN chown` or `RUN chmod` patterns that operate on files that were just copied. If a `COPY` instruction is followed by a `RUN chown -R user:group <same-path>`, flag it and suggest using `COPY --chown=user:group` instead.

**HD008 -- Explicit EXPOSE Only (Low)**
Check that at least one `EXPOSE` instruction exists to document the ports the application listens on. If no `EXPOSE` is present, flag it as undocumented. Do not flag the presence of EXPOSE itself as a problem -- only flag missing EXPOSE or clearly unnecessary extra ports.

### STIG Tier (ST prefix)

**ST001 -- Rootfs-Builder Pattern (High)**
Look for the `--installroot` flag in any `dnf install` or `yum install` command, indicating the rootfs-builder pattern. This is primarily relevant for RHEL/UBI-based images.

**ST002 -- Core Dumps Disabled (Medium)**
Search for `* hard core 0` in the Dockerfile content (typically written to `/etc/security/limits.d/` or `/etc/security/limits.conf`).

**ST003 -- Restrictive Umask 077 (Medium)**
Search for `UMASK 077` or `umask 077` being configured, typically via `sed` on `/etc/login.defs` or an `echo` to a profile script.

**ST004 -- No Empty Passwords (Medium)**
Search for removal of `nullok` from PAM configuration files, typically via `sed` operating on `/etc/pam.d/system-auth` or `/etc/pam.d/password-auth`.

**ST005 -- Max Concurrent Login Sessions (Low)**
Search for `maxlogins` configuration, typically `* hard maxlogins 10` written to `/etc/security/limits.d/`.

**ST006 -- GPG Checking for Packages (Medium)**
Search for `localpkg_gpgcheck=1` in dnf configuration or equivalent GPG verification settings.

**ST007 -- FIPS-Grade Crypto Config (High)**
Search for FIPS crypto policy configuration: `update-crypto-policies --set FIPS`, a `fips_sect` OpenSSL configuration, or crypto-policies-scripts being installed.

**ST008 -- Clean Machine-ID (Low)**
Search for `truncate -s 0 /etc/machine-id`, removal of `/var/lib/dbus/machine-id`, or similar machine-id cleaning operations.

**ST009 -- Scanner Integration Stages (Medium)**
Look for build stages that reference vulnerability scanners: `trivy`, `grype`, `snyk`, `aquasec/trivy`, `anchore/grype`. These are typically separate stages that scan the runtime image.

**ST010 -- Build Metadata LABELs (Low)**
Search for `LABEL` instructions that include OCI-standard metadata keys: `org.opencontainers.image.title`, `org.opencontainers.image.version`, `org.opencontainers.image.created`, `org.opencontainers.image.revision`, `org.opencontainers.image.source`. At minimum, version and VCS reference should be present.

**ST011 -- Minimal Langpack Usage (Low)**
Search for `glibc-minimal-langpack` in package install commands. Flag if full langpacks (`glibc-langpack-*`) are installed instead of the minimal variant. Primarily relevant for RHEL/UBI-based images.

## Step 5: Score the Dockerfile

Calculate a numeric score starting at 100:

- **Critical finding:** deduct 15 points each
- **High finding:** deduct 10 points each
- **Medium finding:** deduct 5 points each
- **Low finding:** deduct 2 points each

The minimum score is 0. Do not go negative.

Determine the highest tier fully met:

| Tier | Requirement |
|------|-------------|
| **STIG** | All SD rules pass AND all HD rules pass AND all ST rules pass |
| **Hardened** | All SD rules pass AND all HD rules pass |
| **Standard** | All SD rules pass |
| **Below Standard** | Any SD rule fails |

A rule that is genuinely not applicable (e.g., ST011 for an Alpine-based image, ST001 for a non-RHEL image) does not count as a failure for tier assessment. Mark it "N/A" internally but do not deduct points or count it against the tier.

## Step 6: Format Output

Always output the following to the conversation:

```
## Dockerfile Audit Results

**File:** `<path/to/Dockerfile>`
**Overall Score: XX/100** | **Tier: [Standard|Hardened|STIG|Below Standard]**

### Critical (N)
- **Line X:** [finding description] -- [remediation]

### High (N)
- **Line X:** [finding description] -- [remediation]

### Medium (N)
- **Line X:** [finding description] -- [remediation]

### Low (N)
- **Line X:** [finding description] -- [remediation]

### Summary
- Total findings: N (X critical, X high, X medium, X low)
- Current tier: [tier]
- To reach [next tier]: [specific changes needed]
```

Formatting rules:
- Only show severity sections that have at least one finding. Omit empty sections entirely.
- Reference specific line numbers wherever possible. For file-level findings (e.g., missing HEALTHCHECK, missing .dockerignore), state "File-level" instead of a line number.
- Keep finding descriptions factual and concise. State the problem and the fix.
- In the "To reach [next tier]" line, list the specific rule IDs and what needs to change. If the Dockerfile already meets STIG tier, say "All tiers met."

## Step 7: Optional Markdown Report

If the `--report` flag was provided:

1. Write the exact same audit results content to `dockerfile-audit.md` in the project root directory.
2. Tell the user: `Report saved to dockerfile-audit.md`

## Step 8: Optional SARIF Output

If the `--sarif` flag was provided:

1. Read the SARIF format template from `${CLAUDE_PLUGIN_ROOT}/knowledge/sarif-format.md`.
2. Generate a valid SARIF 2.1.0 JSON document following the structure and templates defined in that file:
   - Include only rules that produced findings in `tool.driver.rules`.
   - Map severities: Critical/High to `"error"`, Medium to `"warning"`, Low to `"note"`.
   - Include `locations` with line numbers for line-specific findings. Omit `locations` for file-level findings.
   - Use the Dockerfile's path relative to the project root in `artifactLocation.uri`.
   - Use 2-space JSON indentation.
3. Write the JSON to `dockerfile-audit.sarif` in the project root directory.
4. Tell the user: `SARIF report saved to dockerfile-audit.sarif`

## Important Rules for This Skill

- **Be thorough.** Check every single one of the 29 rules. Do not skip rules or short-circuit the analysis.
- **Be specific.** Reference line numbers, quote the problematic instruction, and name the exact issue.
- **Be actionable.** Every finding must include a concrete remediation. Show what to change, not just what is wrong.
- **Do not be preachy.** State facts. Do not lecture about security philosophy or add unnecessary warnings.
- **Handle N/A rules correctly.** STIG rules on a simple Alpine-based Dockerfile, or RHEL-specific rules on Debian images, should be noted as not applicable in tier assessment but not reported as findings.
- **Check the filesystem for SD005.** The `.dockerignore` check requires checking whether the file exists on disk, not just analyzing the Dockerfile content. Use Glob to check.
- **Analyze only the final stage for runtime rules.** Rules like HD001 (USER), HD003 (no shell), HD006 (no package manager), and SD010 (minimal base) apply to the final stage of a multi-stage build, not builder stages.
