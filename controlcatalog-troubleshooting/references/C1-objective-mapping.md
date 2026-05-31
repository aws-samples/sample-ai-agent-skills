---
title: "C1 — Objective Mapping"
description: "Diagnose control-to-objective mapping issues"
status: active
severity: LOW
triggers: ["objective mapping", "which controls for objective", "control objective"]
owner: devops-agent
objective: "Resolve objective mapping issues"
context: "Controls map to objectives (specific compliance goals). Understanding mappings helps select relevant controls."
---

## Phase 1 — Triage

MUST:
- List objectives: `aws controlcatalog list-objectives`
- Check control-objective mappings in control details
- Verify objective ARN is valid

SHOULD:
- Map objectives to your compliance requirements
- Identify controls per objective

MAY:
- Compare objectives across domains

## Phase 2 — Remediate

MUST:
- Use valid objective ARNs for filtering
- Understand objective-control relationships

SHOULD:
- Document objective-to-requirement mappings

MAY:
- Create compliance matrix from catalog data

## Common Issues

- symptoms: "Cannot find controls for a specific compliance objective"
  diagnosis: "Objective ARN incorrect or no controls mapped to that objective."
  resolution: "List objectives to get valid ARNs. Filter controls by objective."

## Output Format

```yaml
root_cause: "objective_mapping — <specific_cause>"
evidence:
  - type: mapping_details
    content: "<objective-control mappings>"
severity: LOW
mitigation:
  immediate: "Fix mapping query"
  long_term: "Maintain compliance mapping documentation"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Objective mappings are public reference data |
| Blast Radius | LOW — Read-only mapping queries |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | HIGH — Incorrect objective-control mappings can create compliance gaps |

## Escalation Conditions

- Escalate to compliance team if objective mappings don't cover required compliance requirements
- Escalate to auditors if mapping gaps are discovered during compliance assessment

## Data Sensitivity

- **Objective and control mappings** are public AWS reference data
- **Compliance requirement mappings** created from catalog data may be organization-sensitive

## PROHIBITED Actions

- Do NOT rely solely on catalog mappings for compliance — supplement with framework-specific guidance
- Do NOT assume catalog mappings are exhaustive for any compliance framework

## Phase 3 — Rollback

- No rollback needed — read-only query
- If compliance decisions were based on incorrect mappings, reassess and update compliance documentation

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
