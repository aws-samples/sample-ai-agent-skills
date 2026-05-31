---
title: "B3 — Spot Interruptions"
description: "Diagnose and handle Spot instance interruptions"
status: active
severity: MEDIUM
triggers:
  - "Spot interrupted"
  - "Spot reclaimed"
  - "instance terminated"
  - "Spot capacity"
  - "SPOT_INSTANCE_RECLAMATION"
owner: devops-agent
objective: "Handle Spot interruptions and improve resilience"
context: "Spot interruptions are normal. Batch can retry interrupted jobs. Issues include: no retry strategy, all Spot (no On-Demand fallback), or insufficient instance type diversity."
---

## Phase 1 — Triage

MUST:
- Check job status reason for Spot interruption: `aws batch describe-jobs --jobs <job-id> --query 'jobs[*].statusReason'`
- Check retry strategy: `aws batch describe-job-definitions --job-definitions <def-name> --query 'jobDefinitions[*].retryStrategy'`
- Check CE allocation strategy: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].computeResources.allocationStrategy'`
- Check instance type diversity in CE

SHOULD:
- Check Spot interruption frequency
- Verify retry strategy handles Spot interruptions
- Check if On-Demand fallback is configured

MAY:
- Check Spot price history for configured instance types
- Review Spot interruption patterns by instance type

## Phase 2 — Remediate

MUST:
- Configure retry strategy for Spot interruptions: `aws batch register-job-definition --job-definition-name <name> --type container --retry-strategy '{"attempts":3,"evaluateOnExit":[{"onStatusReason":"Host EC2*","action":"RETRY"},{"onExitCode":"*","action":"EXIT"}]}' --container-properties '...'`
- Add instance type diversity (4+ types recommended)
- Use SPOT_CAPACITY_OPTIMIZED allocation strategy

SHOULD:
- Mix Spot and On-Demand with separate CEs and queue priority
- Implement checkpointing for long-running jobs

MAY:
- Use Spot placement score to find optimal AZs
- Implement graceful shutdown handling (2-minute warning)

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Configure retry strategy for Spot interruptions | GREEN | Adds resilience — non-destructive |
| Add instance type diversity | GREEN | CE configuration improvement — non-destructive |
| Use SPOT_CAPACITY_OPTIMIZED allocation | GREEN | Optimization — non-destructive |
| Mix Spot and On-Demand with separate CEs | YELLOW | Architecture change — verify queue priority |
| Implement checkpointing for long-running jobs | GREEN | Adds resilience — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Spot interruptions cause repeated job failures affecting SLA
- All-Spot configuration with no On-Demand fallback
- Long-running jobs without checkpointing lose significant progress

## Data Sensitivity

- **Classification: MEDIUM**
- Spot interruption patterns reveal cost optimization strategy
- Retry strategy configuration exposes fault tolerance approach
- Instance type diversity reveals compute requirements
- Allocation strategy reveals infrastructure preferences

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest using only Spot instances for critical production jobs without On-Demand fallback
- **NEVER** suggest disabling retry strategies to avoid Spot interruption retries
- **NEVER** suggest ignoring Spot interruption patterns without implementing resilience

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Configured retry strategy | Remove retry strategy if it causes excessive retries |
| Changed allocation strategy | Revert to previous allocation strategy |
| Created separate On-Demand CE | Delete On-Demand CE if not needed |
| Changed instance type diversity | Revert to previous instance type list |

## Output Format

```yaml
root_cause: "spot_interruption — <specific_cause>"
evidence:
  - type: interruption_reason
    content: "<job status reason>"
  - type: retry_strategy
    content: "<retry configuration>"
severity: MEDIUM
mitigation:
  immediate: "Configure retry strategy and add instance diversity"
  long_term: "Implement Spot best practices with On-Demand fallback"
```

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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"
