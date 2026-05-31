---
title: "A3 — Job Timeout"
description: "Diagnose job timeout issues"
status: active
severity: MEDIUM
triggers:
  - "job timeout"
  - "attempt duration exceeded"
  - "job killed timeout"
  - "job running too long"
  - "timeout exceeded"
owner: devops-agent
objective: "Identify and resolve job timeout issues"
context: "Job timeouts occur when: attemptDurationSeconds is set too low, job processing takes longer than expected, resource contention slows execution, or external dependencies cause delays."
---

## Phase 1 — Triage

MUST:
- Check job timeout setting: `aws batch describe-jobs --jobs <job-id> --query 'jobs[*].timeout'`
- Check job definition timeout: `aws batch describe-job-definitions --job-definitions <def-name> --query 'jobDefinitions[*].timeout'`
- Check job duration vs timeout: compare startedAt and stoppedAt
- Check container logs for slow operations

SHOULD:
- Check resource utilization (CPU, memory) during execution
- Verify external dependencies are responsive
- Check for I/O bottlenecks

MAY:
- Compare execution time across recent job runs
- Check for resource contention on shared instances

## Phase 2 — Remediate

MUST:
- Increase attemptDurationSeconds if job legitimately needs more time
- Optimize job code if execution is unnecessarily slow
- Fix external dependency issues

SHOULD:
- Set appropriate timeout: `aws batch register-job-definition --job-definition-name <name> --type container --timeout '{"attemptDurationSeconds":7200}' --container-properties '...'`
- Implement progress logging for long-running jobs

MAY:
- Break long jobs into smaller array job children
- Implement checkpointing for resumable jobs

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase attemptDurationSeconds | GREEN | Job definition change — non-destructive |
| Optimize job code for faster execution | GREEN | Code improvement — non-destructive |
| Fix external dependency issues | GREEN | Infrastructure fix — non-destructive |
| Break long jobs into array job children | YELLOW | Architecture change — verify data partitioning |
| Implement checkpointing | GREEN | Adds resilience — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Job timeout affects downstream data pipelines or SLA
- Timeout is caused by external dependency failure
- Repeated timeouts indicate systemic performance issue

## Data Sensitivity

- **Classification: MEDIUM**
- Job timeout configuration reveals expected processing duration
- Container logs may contain application output with sensitive data
- Job duration patterns reveal workload characteristics
- External dependency details expose integration architecture

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest setting timeout to 0 (infinite) for production jobs
- **NEVER** suggest killing timed-out jobs without checking for partial output cleanup
- **NEVER** suggest removing timeout entirely as a fix

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Increased attemptDurationSeconds | Reduce timeout to previous value if jobs should not run longer |
| Registered new job definition with timeout | Submit jobs using previous revision |
| Broke job into array children | Consolidate back to single job if partitioning causes issues |

## Output Format

```yaml
root_cause: "job_timeout — <specific_cause>"
evidence:
  - type: timeout_config
    content: "<timeout setting>"
  - type: job_duration
    content: "<actual execution time>"
severity: MEDIUM
mitigation:
  immediate: "Increase timeout or optimize job execution"
  long_term: "Implement job monitoring and optimization"
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
