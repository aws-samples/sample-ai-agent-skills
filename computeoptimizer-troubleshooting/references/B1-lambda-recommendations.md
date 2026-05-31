---
title: "B1 — Lambda Function Recommendations"
description: "Diagnose missing or incorrect Lambda recommendations"
status: active
severity: MEDIUM
triggers: ["Lambda recommendation", "function memory", "Lambda sizing"]
owner: devops-agent
objective: "Resolve Lambda recommendation issues"
context: "Lambda recommendations optimize memory allocation based on invocation patterns."
---

## Phase 1 — Triage

MUST:
- Get recommendations: `aws compute-optimizer get-lambda-function-recommendations --function-arns <arn>`
- Verify function has sufficient invocations
- Check enrollment status

SHOULD:
- Check function invocation frequency
- Verify function has 14+ days of data

## Phase 2 — Remediate

MUST:
- Enroll if not opted in
- Wait for sufficient invocation data

## Common Issues

- symptoms: "No Lambda recommendations"
  diagnosis: "Function has too few invocations or is too new."
  resolution: "Functions need regular invocations over 14+ days."

## Output Format

```yaml
root_cause: "lambda_recommendations — <specific_cause>"
evidence:
  - type: recommendation
    content: "<Lambda recommendation details>"
severity: MEDIUM
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Recommendations based on invocation metrics, not function data |
| Blast Radius | MEDIUM — Memory changes affect CPU allocation and may impact function performance |
| Rollback Complexity | LOW — Lambda memory can be changed instantly without downtime |
| Compliance Scope | LOW — Advisory recommendations only |

## Escalation Conditions

- Escalate to function owners before changing memory allocation on production functions
- Escalate to operations if memory reduction causes function timeouts or errors

## Data Sensitivity

- **Function ARNs and invocation metrics** are standard operational data
- **Recommendation details** do not contain function code or payload data

## PROHIBITED Actions

- Do NOT reduce Lambda memory on production functions without testing the impact on duration and errors
- Do NOT apply memory changes to all functions simultaneously — use staged rollout
- Do NOT ignore duration increases after memory reduction — they indicate CPU-bound workloads

## Phase 3 — Rollback

- Revert Lambda memory to previous value: `aws lambda update-function-configuration --function-name <name> --memory-size <previous-value>`
- Monitor function duration and error rate after rollback
- Document the failed optimization for future reference

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
