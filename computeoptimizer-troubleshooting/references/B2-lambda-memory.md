---
title: "B2 — Lambda Memory Sizing Issues"
description: "Diagnose Lambda memory optimization recommendations"
status: active
severity: LOW
triggers: ["Lambda memory", "over-provisioned Lambda", "under-provisioned Lambda"]
owner: devops-agent
objective: "Resolve Lambda memory sizing issues"
context: "Lambda memory affects CPU allocation and cost."
---

## Phase 1 — Triage

MUST:
- Review memory recommendation details
- Check current vs recommended memory
- Verify invocation duration patterns

## Phase 2 — Remediate

MUST:
- Test with recommended memory setting
- Monitor duration after change

## Common Issues

- symptoms: "Recommendation suggests lower memory but function is CPU-bound"
  diagnosis: "Recommendation based on memory utilization, not CPU."
  resolution: "Test with recommended memory. If duration increases, keep current."

## Output Format

```yaml
root_cause: "lambda_memory — <specific_cause>"
evidence:
  - type: memory_analysis
    content: "<memory recommendation>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Memory sizing analysis uses invocation metrics only |
| Blast Radius | MEDIUM — Incorrect memory sizing affects function performance and cost |
| Rollback Complexity | LOW — Memory changes are instant and non-destructive |
| Compliance Scope | LOW — No compliance impact |

## Escalation Conditions

- Escalate to function owners if memory reduction causes performance degradation
- Escalate to operations if under-provisioned memory causes OOM errors

## Data Sensitivity

- **Memory utilization data** is standard operational metrics
- No application data is exposed through memory sizing analysis

## PROHIBITED Actions

- Do NOT reduce memory below the minimum required for the function's dependencies
- Do NOT change memory on latency-sensitive functions without load testing

## Phase 3 — Rollback

- Revert to previous memory setting immediately via Lambda configuration update
- Monitor function CloudWatch metrics (Duration, Errors, Throttles) after rollback

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
