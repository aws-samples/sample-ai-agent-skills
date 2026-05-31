---
title: "C2 — IOPS Optimization Issues"
description: "Diagnose EBS IOPS and throughput optimization"
status: active
severity: LOW
triggers: ["IOPS too high", "throughput recommendation", "volume performance"]
owner: devops-agent
objective: "Resolve IOPS optimization issues"
context: "IOPS and throughput recommendations depend on workload patterns."
---

## Phase 1 — Triage

MUST:
- Review IOPS utilization metrics
- Check throughput utilization
- Compare current vs recommended IOPS

## Phase 2 — Remediate

MUST:
- Adjust IOPS based on recommendation
- Monitor after changes

## Common Issues

- symptoms: "Over-provisioned IOPS on io2 volume"
  diagnosis: "Provisioned IOPS much higher than actual usage."
  resolution: "Reduce provisioned IOPS or switch to gp3."

## Output Format

```yaml
root_cause: "iops_optimization — <specific_cause>"
evidence:
  - type: iops_analysis
    content: "<IOPS utilization>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — IOPS analysis uses volume performance metrics only |
| Blast Radius | MEDIUM — IOPS changes directly affect storage performance |
| Rollback Complexity | MEDIUM — IOPS changes have a 6-hour cooldown between modifications |
| Compliance Scope | LOW — No compliance impact |

## Escalation Conditions

- Escalate to database/application owners before reducing provisioned IOPS
- Escalate to operations if IOPS reduction causes latency spikes

## Data Sensitivity

- **IOPS utilization data** is standard infrastructure metrics
- No application data is exposed

## PROHIBITED Actions

- Do NOT reduce IOPS on database volumes without load testing
- Do NOT make IOPS changes without accounting for the 6-hour modification cooldown

## Phase 3 — Rollback

- Modify IOPS back to previous value (subject to cooldown period)
- If urgent, create a new volume from snapshot with original IOPS configuration and swap
- Monitor I/O latency metrics after rollback

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
