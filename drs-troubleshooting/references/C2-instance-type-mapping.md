---
title: "C2 — Instance Type Mapping"
description: "Diagnose instance type right-sizing and mapping issues"
status: active
severity: MEDIUM
triggers:
  - "wrong instance type"
  - "instance type mapping"
  - "right-sizing"
owner: devops-agent
objective: "Resolve instance type mapping for recovery instances"
context: "DRS can auto-select instance types based on source server specs (right-sizing) or use explicitly configured types. Mapping issues cause undersized or oversized recovery instances."
---

## Phase 1 — Triage

MUST:
- Check right-sizing method: `aws drs get-launch-configuration --source-server-id <id> --query 'targetInstanceTypeRightSizingMethod'`
- Compare source server specs with target instance type
- Verify target instance type exists in the target region

SHOULD:
- Check if BASIC or NONE right-sizing is configured
- Verify instance type supports required features (ENA, NVMe)

MAY:
- Review EC2 instance type availability in target AZ

## Phase 2 — Remediate

MUST:
- Set explicit instance type if auto-sizing selects wrong type: `aws drs update-launch-configuration --source-server-id <id> --target-instance-type-right-sizing-method NONE`
- Verify selected instance type meets application requirements

SHOULD:
- Test with drill launch before actual recovery
- Document instance type decisions for each source server

MAY:
- Use BASIC right-sizing for initial mapping, then fine-tune

## Common Issues

- symptoms: "Recovery instance too small for application workload"
  diagnosis: "Right-sizing selected based on CPU/RAM but not I/O or network requirements."
  resolution: "Override with explicit instance type matching application needs."

## Output Format

```yaml
root_cause: "instance_type_mapping — <specific_cause>"
evidence:
  - type: right_sizing
    content: "<right-sizing method and selected type>"
severity: MEDIUM
mitigation:
  immediate: "Set correct instance type"
  long_term: "Document and validate instance type mappings for all source servers"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Instance type mapping is operational configuration |
| Blast Radius | MEDIUM — Incorrect mapping causes undersized or oversized recovery instances |
| Rollback Complexity | LOW — Launch configuration can be updated |
| Compliance Scope | MEDIUM — Correct sizing ensures application performance during DR |

## Escalation Conditions

- Escalate to application owners if right-sizing selects inappropriate instance types
- Escalate to DR coordinator if mapping issues affect DR readiness

## Data Sensitivity

- **Instance type mappings** are operational configuration
- **Source server specifications** reveal infrastructure details

## PROHIBITED Actions

- Do NOT use BASIC right-sizing for applications with specific hardware requirements
- Do NOT deploy instance type changes without drill testing

## Phase 3 — Rollback

- Restore previous instance type in launch configuration
- Validate with a drill test after changes

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
