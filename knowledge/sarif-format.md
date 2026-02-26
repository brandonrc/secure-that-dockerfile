# SARIF 2.1.0 Output Format

## Overview

SARIF (Static Analysis Results Interchange Format) is an OASIS standard (SARIF 2.1.0) for representing static analysis results as structured JSON. It is consumed by:

- **GitHub Code Scanning** -- upload via `github/codeql-action/upload-sarif` to see findings as code annotations
- **VS Code SARIF Viewer** -- the `MS-SarifVSCode.sarif-viewer` extension renders results inline in the editor
- **Azure DevOps** -- native SARIF tab in pipeline results
- **Defect Dojo, SonarQube, and other aggregators** -- import SARIF for centralized vulnerability tracking

When the audit skill runs with `--sarif`, it reads this file to produce a valid SARIF 2.1.0 JSON document written to `dockerfile-audit.sarif`.

---

## SARIF Document Structure

Every SARIF file produced by this plugin follows this top-level structure:

```json
{
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "secure-dockerfile",
          "version": "0.1.0",
          "informationUri": "https://github.com/brandonrc/secure-that-dockerfile",
          "rules": [
            // One entry per rule that produced at least one finding
          ]
        }
      },
      "results": [
        // One entry per finding
      ]
    }
  ]
}
```

Key points:
- There is always exactly **one run** in the `runs` array.
- The `tool.driver.rules` array contains only the rules that triggered findings in this audit (not all 29 rules).
- The `results` array contains one entry per finding, referencing a rule by `ruleId`.

---

## Severity Mapping

The plugin uses five severity levels. They map to SARIF `level` values as follows:

| Plugin Severity | SARIF Level | When Used |
|-----------------|-------------|-----------|
| Critical        | error       | Vulnerabilities that must be fixed immediately (secrets in image, unpinned tags, running as root) |
| High            | error       | Significant security gaps (no multi-stage, no capability drops, no FIPS) |
| Medium          | warning     | Best practice violations that weaken security posture |
| Low             | note        | Improvements that reduce image size or improve maintainability |
| Info            | note        | Informational suggestions with no direct security impact |

---

## Rule Definition Template

Each rule from `security-tiers.md` maps to a SARIF rule object in `tool.driver.rules`. Use this template:

```json
{
  "id": "SD001",
  "name": "UnpinnedBaseImageTag",
  "shortDescription": {
    "text": "Base image uses unpinned tag"
  },
  "fullDescription": {
    "text": "Base images should use specific version tags or digests, not 'latest' or untagged references. Unpinned tags cause non-reproducible builds and may pull in unvetted changes."
  },
  "helpUri": "https://github.com/brandonrc/secure-that-dockerfile#SD001",
  "defaultConfiguration": {
    "level": "error"
  },
  "properties": {
    "tier": "standard",
    "category": "security"
  }
}
```

Field notes:
- `id` -- the rule ID from security-tiers.md (e.g., `SD001`, `HD003`, `ST007`)
- `name` -- PascalCase descriptive name (unique across all rules)
- `shortDescription.text` -- one-line summary (under 120 characters)
- `fullDescription.text` -- detailed explanation with rationale
- `helpUri` -- link to the rule documentation
- `defaultConfiguration.level` -- the SARIF level derived from the severity mapping table above
- `properties.tier` -- which tier the rule belongs to: `standard`, `hardened`, or `stig`
- `properties.category` -- always `security` for this plugin

---

## Result Template

Each finding in the `results` array follows this template:

```json
{
  "ruleId": "SD001",
  "level": "error",
  "message": {
    "text": "Base image 'ubuntu:latest' uses unpinned tag. Pin to a specific version like 'ubuntu:24.04' or use a digest."
  },
  "locations": [
    {
      "physicalLocation": {
        "artifactLocation": {
          "uri": "Dockerfile",
          "uriBaseId": "%SRCROOT%"
        },
        "region": {
          "startLine": 1,
          "startColumn": 1,
          "endLine": 1
        }
      }
    }
  ]
}
```

Field notes:
- `ruleId` -- must match a rule `id` in the `tool.driver.rules` array
- `level` -- must match the severity mapping (can override the rule default if contextually appropriate, but typically matches)
- `message.text` -- human-readable finding with the specific problematic value and a concrete remediation suggestion
- `locations[].physicalLocation.artifactLocation.uri` -- relative path to the Dockerfile from the project root
- `locations[].physicalLocation.artifactLocation.uriBaseId` -- always `%SRCROOT%`
- `locations[].physicalLocation.region.startLine` -- the 1-based line number in the Dockerfile where the issue occurs
- `locations[].physicalLocation.region.startColumn` -- typically `1` unless the issue is at a specific column
- `locations[].physicalLocation.region.endLine` -- the last line of the problematic instruction

For findings that are not tied to a specific line (e.g., "no HEALTHCHECK defined", "no .dockerignore"), omit the `locations` array entirely.

---

## Complete Rule ID Reference Table

This is the lookup table the audit skill uses to map every rule to its SARIF representation.

### Standard Tier (SD001 -- SD010)

| Rule ID | SARIF `name`            | Short Description                          | Severity | SARIF `level` |
|---------|-------------------------|--------------------------------------------|----------|---------------|
| SD001   | UnpinnedBaseImageTag    | Base image uses unpinned tag               | Critical | error         |
| SD002   | NoMultiStageBuild       | No multi-stage build detected              | High     | error         |
| SD003   | SeparateRunLayers       | RUN commands not combined                  | Medium   | warning       |
| SD004   | MissingCacheCleanup     | Package cache not cleaned in same layer    | Medium   | warning       |
| SD005   | MissingDockerignore     | No .dockerignore file present              | Medium   | warning       |
| SD006   | SecretsInImage          | Secrets exposed in ENV, ARG, or COPY       | Critical | error         |
| SD007   | MissingHealthcheck      | No HEALTHCHECK instruction defined         | Medium   | warning       |
| SD008   | BroadCopySource         | COPY uses broad source (COPY . .)          | Low      | note          |
| SD009   | PoorLayerOrdering       | Source copied before dependency install     | Low      | note          |
| SD010   | LargeBaseImage          | Base image is not minimal variant           | Medium   | warning       |

### Hardened Tier (HD001 -- HD008)

| Rule ID | SARIF `name`                | Short Description                              | Severity | SARIF `level` |
|---------|-----------------------------|-------------------------------------------------|----------|---------------|
| HD001   | RunningAsRoot               | No non-root USER directive                      | Critical | error         |
| HD002   | WritableFilesystem          | Not read-only filesystem friendly               | Medium   | warning       |
| HD003   | ShellInFinalImage           | Shell present in final image                    | Medium   | warning       |
| HD004   | UnnecessaryPackages         | Packages installed without --no-install-recommends | Medium   | warning       |
| HD005   | MissingCapabilityDrops      | No guidance to drop all capabilities            | High     | error         |
| HD006   | PackageManagerInRuntime     | Package manager present in runtime image        | Medium   | warning       |
| HD007   | RunChownInsteadOfCopyChown  | RUN chown used instead of COPY --chown          | Low      | note          |
| HD008   | UndocumentedExpose          | EXPOSE used without documentation or unnecessary ports exposed | Low      | note          |

### STIG Tier (ST001 -- ST011)

| Rule ID | SARIF `name`              | Short Description                              | Severity | SARIF `level` |
|---------|---------------------------|-------------------------------------------------|----------|---------------|
| ST001   | NoRootfsBuilderPattern    | No rootfs-builder pattern (--installroot)       | High     | error         |
| ST002   | CoreDumpsEnabled          | Core dumps not disabled                         | Medium   | warning       |
| ST003   | PermissiveUmask           | Umask not set to 077                            | Medium   | warning       |
| ST004   | EmptyPasswordsAllowed     | Empty passwords not explicitly prevented        | Medium   | warning       |
| ST005   | NoConcurrentSessionLimit  | No max concurrent login sessions configured     | Low      | note          |
| ST006   | MissingGpgCheck           | GPG checking not enabled for packages           | Medium   | warning       |
| ST007   | NoFipsCryptoConfig        | FIPS-grade crypto not configured                | High     | error         |
| ST008   | StaleMachineId            | machine-id not cleaned for runtime regeneration | Low      | note          |
| ST009   | NoScannerIntegration      | No vulnerability scanner stage in build         | Medium   | warning       |
| ST010   | MissingBuildMetadata      | No build metadata LABELs present                | Low      | note          |
| ST011   | FullLangpackInstalled     | Full langpack installed instead of minimal       | Low      | note          |

---

## Complete Example

The following is a complete SARIF output for a sample Dockerfile with four findings. The audit skill should produce output matching this structure exactly.

### Sample Dockerfile (input)

```dockerfile
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y python3 python3-pip

COPY . .

ENV DATABASE_URL=postgres://admin:secret@db:5432/myapp

RUN pip3 install -r requirements.txt

EXPOSE 8000

CMD ["python3", "app.py"]
```

### SARIF Output (dockerfile-audit.sarif)

```json
{
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "secure-dockerfile",
          "version": "0.1.0",
          "informationUri": "https://github.com/brandonrc/secure-that-dockerfile",
          "rules": [
            {
              "id": "SD001",
              "name": "UnpinnedBaseImageTag",
              "shortDescription": {
                "text": "Base image uses unpinned tag"
              },
              "fullDescription": {
                "text": "Base images should use specific version tags or digests, not 'latest' or untagged references. Unpinned tags cause non-reproducible builds and may pull in unvetted changes."
              },
              "helpUri": "https://github.com/brandonrc/secure-that-dockerfile#SD001",
              "defaultConfiguration": {
                "level": "error"
              },
              "properties": {
                "tier": "standard",
                "category": "security"
              }
            },
            {
              "id": "SD003",
              "name": "SeparateRunLayers",
              "shortDescription": {
                "text": "RUN commands not combined"
              },
              "fullDescription": {
                "text": "apt-get update and apt-get install should be combined in a single RUN instruction to prevent stale cache issues and reduce image layers."
              },
              "helpUri": "https://github.com/brandonrc/secure-that-dockerfile#SD003",
              "defaultConfiguration": {
                "level": "warning"
              },
              "properties": {
                "tier": "standard",
                "category": "security"
              }
            },
            {
              "id": "SD006",
              "name": "SecretsInImage",
              "shortDescription": {
                "text": "Secrets exposed in ENV, ARG, or COPY"
              },
              "fullDescription": {
                "text": "Sensitive values such as passwords, API keys, and connection strings with credentials must not be stored in ENV or ARG instructions. They are visible in the image history and layer metadata."
              },
              "helpUri": "https://github.com/brandonrc/secure-that-dockerfile#SD006",
              "defaultConfiguration": {
                "level": "error"
              },
              "properties": {
                "tier": "standard",
                "category": "security"
              }
            },
            {
              "id": "HD001",
              "name": "RunningAsRoot",
              "shortDescription": {
                "text": "No non-root USER directive"
              },
              "fullDescription": {
                "text": "Containers should not run as root. Create a dedicated non-root user and switch to it with the USER instruction to limit the blast radius of a container compromise."
              },
              "helpUri": "https://github.com/brandonrc/secure-that-dockerfile#HD001",
              "defaultConfiguration": {
                "level": "error"
              },
              "properties": {
                "tier": "hardened",
                "category": "security"
              }
            }
          ]
        }
      },
      "results": [
        {
          "ruleId": "SD001",
          "level": "error",
          "message": {
            "text": "Base image 'ubuntu:latest' uses unpinned tag. Pin to a specific version like 'ubuntu:24.04' or use a digest (e.g., ubuntu@sha256:...)."
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "Dockerfile",
                  "uriBaseId": "%SRCROOT%"
                },
                "region": {
                  "startLine": 1,
                  "startColumn": 1,
                  "endLine": 1
                }
              }
            }
          ]
        },
        {
          "ruleId": "SD003",
          "level": "warning",
          "message": {
            "text": "'apt-get update' on line 3 and 'apt-get install' on line 4 are in separate RUN instructions. Combine them into a single RUN layer: 'RUN apt-get update && apt-get install -y --no-install-recommends python3 python3-pip && rm -rf /var/lib/apt/lists/*'."
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "Dockerfile",
                  "uriBaseId": "%SRCROOT%"
                },
                "region": {
                  "startLine": 3,
                  "startColumn": 1,
                  "endLine": 4
                }
              }
            }
          ]
        },
        {
          "ruleId": "SD006",
          "level": "error",
          "message": {
            "text": "ENV 'DATABASE_URL' contains what appears to be a credential ('secret' in connection string). Use Docker BuildKit secrets (--mount=type=secret) or runtime environment injection instead of baking secrets into the image."
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "Dockerfile",
                  "uriBaseId": "%SRCROOT%"
                },
                "region": {
                  "startLine": 8,
                  "startColumn": 1,
                  "endLine": 8
                }
              }
            }
          ]
        },
        {
          "ruleId": "HD001",
          "level": "error",
          "message": {
            "text": "No USER instruction found. The container will run as root. Add a non-root user: 'RUN useradd -r -s /usr/sbin/nologin appuser' and 'USER appuser'."
          }
        }
      ]
    }
  ]
}
```

### What This Example Demonstrates

1. **Rule filtering** -- Only the four rules that produced findings appear in `tool.driver.rules`. The remaining 25 rules are omitted.
2. **Level mapping** -- SD001 (Critical) and SD006 (Critical) map to `error`. SD003 (Medium) maps to `warning`. HD001 (Critical) maps to `error`.
3. **Location specificity** -- Findings tied to specific lines include `locations` with line numbers. HD001 (no USER directive) has no `locations` because it is a file-level finding.
4. **Actionable messages** -- Each `message.text` names the specific problem, quotes the offending value, and provides a concrete fix.

---

## Production Notes

### File Output

When the audit skill writes `dockerfile-audit.sarif`:
- Use 2-space indentation for readability
- Ensure valid JSON (no trailing commas, proper escaping)
- Write to the project root directory alongside the Dockerfile

### GitHub Code Scanning Integration

Users can upload the SARIF file in a GitHub Actions workflow:

```yaml
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: dockerfile-audit.sarif
    category: dockerfile-security
```

### Validation

The SARIF output should validate against the official schema:
- Schema URL: `https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json`
- Online validator: https://sarifweb.azurewebsites.net/Validation
