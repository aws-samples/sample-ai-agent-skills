---
title: "D2 — Provisioned Throughput Issues"
description: "Diagnose Bedrock provisioned throughput problems"
status: active
severity: HIGH
triggers:
  - "provisioned throughput error"
  - "model units"
  - "dedicated capacity"
  - "provisioned model"
owner: devops-agent
objective: "Identify and resolve provisioned throughput issues"
context: "Provisioned throughput provides dedicated model capacity with guaranteed token rates. It's billed even when idle. No auto-scaling. Commitment terms affect pricing. Issues include creation failures, insufficient capacity, and cost management."
---

## Phase 1 — Triage

MUST:
- List provisioned throughputs: `aws bedrock list-provisioned-model-throughputs`
- Check specific throughput: `aws bedrock get-provisioned-model-throughput --provisioned-model-id <id>`
- Verify the base model supports provisioned throughput
- Check throughput status (InService, Creating, Failed)

SHOULD:
- Check CloudWatch metrics for throughput utilization
- Verify the model units are sufficient for the workload
- Check commitment term and expiration
- Review cost implications

MAY:
- Compare provisioned vs on-demand costs
- Check for regional availability

## Phase 2 — Remediate

MUST:
- Ensure the base model supports provisioned throughput
- Create with appropriate model units for the workload
- Verify throughput is InService before using

SHOULD:
- Monitor utilization to right-size capacity
- Delete unused provisioned throughput to stop charges
- Plan commitment terms based on usage patterns

MAY:
- Implement cost monitoring for provisioned throughput
- Create capacity planning automation

## Common Issues

- symptoms: "Provisioned throughput creation fails"
  diagnosis: "Model doesn't support provisioned throughput or capacity unavailable."
  resolution: "Verify model supports provisioned throughput. Try different region."

- symptoms: "Still getting throttled with provisioned throughput"
  diagnosis: "Insufficient model units for the request rate."
  resolution: "Increase model units. Check if using the provisioned model ARN (not base model ID)."

## Output Format

```yaml
root_cause: "provisioned_throughput — <specific_cause>"
evidence:
  - type: throughput_status
    content: "<provisioned throughput configuration>"
  - type: utilization
    content: "<throughput utilization metrics>"
severity: HIGH
mitigation:
  immediate: "Fix throughput configuration"
  long_term: "Implement capacity planning and cost monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure the base model supports provisioned throughput | GREEN |
| Create with appropriate model units for the workload | YELLOW |
| Verify throughput is InService before using | GREEN |
| Monitor utilization to right-size capacity | GREEN |
| Delete unused provisioned throughput to stop charges | RED |
| Plan commitment terms based on usage patterns | YELLOW |
| Implement cost monitoring for provisioned throughput | GREEN |
| Create capacity planning automation | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Provisioned throughput usage patterns may reveal business-critical workload information

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If provisioned throughput was created in error, delete it: `aws bedrock delete-provisioned-model-throughput --provisioned-model-id <id>` (note: commitment-based throughput cannot be deleted before term ends)
2. If applications were switched to use provisioned model ARN, revert to on-demand model ID
3. If model units were changed, update back to the previous model unit count
4. Verify rollback by confirming `aws bedrock list-provisioned-model-throughputs` shows expected state

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
