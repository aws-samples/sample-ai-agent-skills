---
title: "F2 — Export Configuration Issues"
description: "Diagnose export format and scope configuration"
status: active
severity: LOW
triggers: ["export format", "export scope", "export fields"]
owner: devops-agent
objective: "Resolve export configuration issues"
context: "Exports can be configured for specific resource types and fields."
---

## Phase 1 — Triage

MUST:
- Review export configuration
- Check resource type filter
- Verify field selection

## Phase 2 — Remediate

MUST:
- Correct export parameters
- Specify correct resource types

## Common Issues

- symptoms: "Export missing expected resources"
  diagnosis: "Resource type filter too restrictive."
  resolution: "Include all needed resource types in export."

## Output Format

```yaml
root_cause: "export_config — <specific_cause>"
evidence:
  - type: export_config
    content: "<export configuration>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Export configuration is operational metadata |
| Blast Radius | LOW — Configuration changes only affect future exports |
| Rollback Complexity | LOW — Reconfigure and re-export |
| Compliance Scope | LOW — No compliance impact |

## Escalation Conditions

- Escalate to report consumers if export format or scope changes affect their workflows
- Escalate to operations if missing resource types in exports cause planning gaps

## Data Sensitivity

- **Export configuration** is operational metadata — low sensitivity
- **Exported fields** may include resource identifiers — apply standard access controls

## PROHIBITED Actions

- Do NOT change export configuration without notifying downstream report consumers
- Do NOT export to S3 buckets without encryption enabled

## Phase 3 — Rollback

- Revert export configuration to previous settings
- Re-export with corrected parameters
- Notify consumers of any data gaps from the misconfiguration period

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
