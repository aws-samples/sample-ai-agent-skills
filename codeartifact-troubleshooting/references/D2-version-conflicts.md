---
title: "D2 — Version Conflicts"
description: "Diagnose CodeArtifact package version conflicts including immutability and upstream shadowing"
status: active
severity: MEDIUM
triggers:
  - "version conflict"
  - "ConflictException"
  - "version already exists"
  - "wrong version resolved"
  - "dependency confusion"
owner: devops-agent
objective: "Resolve package version conflicts and ensure correct version resolution"
context: "CodeArtifact package versions are immutable. Version conflicts arise from attempting to overwrite versions, upstream shadowing (dependency confusion), or incorrect version resolution order."
---

## Phase 1 — Triage

MUST:
- Check existing versions: `aws codeartifact list-package-versions --domain <domain> --repository <repo> --package <pkg> --format <format>`
- Check which upstream provided the version: `aws codeartifact describe-package-version --domain <domain> --repository <repo> --package <pkg> --format <format> --package-version <version>`
- Verify upstream repository order for resolution priority

SHOULD:
- Check if the same package name exists in multiple upstreams
- Verify internal packages are not being shadowed by public packages
- Check package origin restrictions: `aws codeartifact describe-package --domain <domain> --repository <repo> --package <pkg> --format <format>`

MAY:
- Set package origin controls to restrict where versions can come from
- Review all upstream repositories for conflicting package names

## Phase 2 — Remediate

MUST:
- If version conflict on publish, use new version number or delete existing
- If dependency confusion, set package origin controls: `aws codeartifact put-package-origin-configuration --domain <domain> --repository <repo> --package <pkg> --format <format> --restrictions publish=ALLOW,upstream=BLOCK`
- Reorder upstreams to prioritize internal packages

SHOULD:
- Use package origin controls to prevent external packages from shadowing internal ones
- Audit package names against public registries

## Output Format

```yaml
root_cause: "version_conflict — <specific_cause>"
evidence:
  - type: version_list
    content: "<package versions>"
  - type: origin_info
    content: "<package origin details>"
severity: MEDIUM
mitigation:
  immediate: "Fix version conflict or origin controls"
  long_term: "Implement package origin restrictions and upstream ordering policy"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use new version number | GREEN |
| Delete existing version | RED |
| Set package origin controls | YELLOW |
| Reorder upstreams to prioritize internal packages | YELLOW |
| Audit package names against public registries | GREEN |

## Escalation Conditions

- Fix requires modifying production package origin controls
- Fix involves reordering upstreams which affects all package resolution

## Data Sensitivity

- **HIGH**: version conflicts may indicate dependency confusion attacks
- Package origin controls are a critical security mechanism against supply chain attacks

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If package origin controls were changed, restore previous restrictions
2. If upstream order was modified, restore previous order
3. If a package version was deleted, publish the correct version
4. Verify rollback by confirming package resolution returns expected versions

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
