---
title: "C2 — Resource Association"
description: "Diagnose resource association failures"
status: active
severity: LOW
triggers: ["cannot associate resource", "association error"]
owner: devops-agent
objective: "Resolve resource association issues"
context: "Resources are associated with OpsItems via ARN. Association requires valid ARN format and IAM permissions."
---

## Phase 1 — Triage

MUST:
- Verify ARN format is valid
- Check IAM permissions for ssm:AssociateOpsItemRelatedItem
- Verify OpsItem exists and is not resolved

SHOULD:
- Check resource exists in the account/region

MAY:
- Review association limits per OpsItem

## Phase 2 — Remediate

MUST:
- Fix ARN format
- Add IAM permissions

SHOULD:
- Validate ARN before association

MAY:
- Automate association via Lambda

## Common Issues

- symptoms: "AssociateOpsItemRelatedItem returns validation error"
  diagnosis: "Invalid ARN format."
  resolution: "Use correct ARN format: arn:aws:service:region:account:resource-type/resource-id"

## Output Format

```yaml
root_cause: "resource_association — <specific_cause>"
evidence:
  - type: association_error
    content: "<error details>"
severity: LOW
mitigation:
  immediate: "Fix association"
  long_term: "Standardize ARN handling"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — resource associations are metadata references
- Service Disruption Risk: LOW — association changes do not affect resources
- Compliance Impact: LOW — resource tracking supports operational visibility

## Escalation Conditions
- Resource associations cannot be created or updated
- Association data becomes stale or inaccurate
- Cross-account associations fail due to permission changes
- Association limits prevent tracking all related resources
- Automated association rules stop functioning

## Data Sensitivity
- Resource identifiers and ARNs — INTERNAL
- Association metadata — INTERNAL
- Cross-account references — INTERNAL
- Automated association rules — INTERNAL
- Never expose internal resource identifiers externally

## Prohibited Actions
- Do NOT create associations to resources in unauthorized accounts
- Do NOT ignore stale resource associations
- Do NOT bypass association validation rules
- Do NOT delete associations during active investigations
- Do NOT create excessive associations that reduce operational clarity
- Do NOT associate resources without verifying their existence

## Phase 3 — Rollback
- Remove incorrectly created associations
- Restore previous association configurations
- Re-enable automated association rules if disabled
- Verify association accuracy after changes
- Document association changes and their rationale
- Update association management procedures

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
