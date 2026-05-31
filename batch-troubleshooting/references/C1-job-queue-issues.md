---
title: "C1 — Job Queue Issues"
description: "Diagnose job queue configuration and state problems"
status: active
severity: MEDIUM
triggers:
  - "job queue disabled"
  - "queue not processing"
  - "queue configuration"
  - "jobs not scheduling"
  - "queue priority"
owner: devops-agent
objective: "Identify and resolve job queue issues"
context: "Job queue issues include: queue disabled, no CE associated, CE in INVALID state, priority misconfigured, or fair-share scheduling conflicts."
---

## Phase 1 — Triage

MUST:
- Check queue status: `aws batch describe-job-queues --job-queues <queue-name> --query 'jobQueues[*].{State:state,Status:status,Priority:priority,CEs:computeEnvironmentOrder}'`
- Verify queue is ENABLED
- Check associated CEs are VALID and ENABLED
- Check pending jobs: `aws batch list-jobs --job-queue <queue-name> --job-status RUNNABLE`

SHOULD:
- Verify CE order in queue (evaluated in order)
- Check for fair-share scheduling policy
- Verify queue priority relative to other queues

MAY:
- Check for scheduling policy conflicts
- Review queue job history

## Phase 2 — Remediate

MUST:
- Enable disabled queue: `aws batch update-job-queue --job-queue <queue-name> --state ENABLED`
- Fix associated CE issues
- Ensure at least one valid CE is associated

SHOULD:
- Set appropriate priority (higher number = higher priority)
- Order CEs by preference (first CE is preferred)

MAY:
- Implement fair-share scheduling for multi-tenant workloads
- Create separate queues for different workload types

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable disabled queue | YELLOW | Enables job processing — verify CE is ready |
| Fix associated CE issues | YELLOW | CE changes may affect running jobs |
| Set appropriate priority | GREEN | Configuration change — non-destructive |
| Order CEs by preference | GREEN | Configuration change — non-destructive |
| Implement fair-share scheduling | YELLOW | Changes resource distribution — affects all users |

## Escalation Conditions

- Compute environment runs production batch jobs
- Queue disabled or misconfigured blocking all job processing
- Priority changes affect other teams' job scheduling
- Fair-share policy changes affect multi-tenant resource allocation

## Data Sensitivity

- **Classification: MEDIUM**
- Queue configuration reveals job processing architecture
- CE association reveals infrastructure allocation strategy
- Priority settings expose workload importance hierarchy
- Fair-share policies reveal multi-tenant resource governance

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest disabling a queue with pending production jobs without notification
- **NEVER** suggest changing queue priority without understanding impact on other queues
- **NEVER** suggest removing CE associations while jobs are in RUNNABLE state

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Enabled disabled queue | Disable queue again if it should not be processing |
| Changed queue priority | Revert priority to previous value |
| Changed CE ordering | Revert CE order to previous configuration |
| Attached fair-share scheduling policy | Detach scheduling policy if not needed |

## Output Format

```yaml
root_cause: "job_queue_issue — <specific_cause>"
evidence:
  - type: queue_config
    content: "<queue state and configuration>"
  - type: ce_association
    content: "<associated compute environments>"
severity: MEDIUM
mitigation:
  immediate: "Enable queue or fix CE association"
  long_term: "Implement queue monitoring and proper CE ordering"
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
