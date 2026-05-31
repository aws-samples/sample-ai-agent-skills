---
title: "F1 — Array Job Issues"
description: "Diagnose array job configuration and execution problems"
status: active
severity: MEDIUM
triggers:
  - "array job failed"
  - "array index"
  - "child job error"
  - "array size"
  - "array dependency"
owner: devops-agent
objective: "Identify and resolve array job issues"
context: "Array job issues include: child jobs failing independently, array index not used correctly, dependency configuration wrong, or array size limits."
---

## Phase 1 — Triage

MUST:
- Check array job status: `aws batch describe-jobs --jobs <parent-job-id> --query 'jobs[*].{Status:status,ArraySize:arrayProperties.size}'`
- List child jobs: `aws batch list-jobs --array-job-id <parent-job-id> --job-status FAILED`
- Check specific child job: `aws batch describe-jobs --jobs <parent-job-id>:<index>`
- Verify AWS_BATCH_JOB_ARRAY_INDEX usage in container code

SHOULD:
- Check child job exit codes and logs
- Verify array size is within limits (2-10,000)
- Check dependency configuration between array jobs

MAY:
- Analyze failure patterns across array indices
- Check for index-specific data issues

## Phase 2 — Remediate

MUST:
- Fix child job failures based on specific error
- Ensure container code uses AWS_BATCH_JOB_ARRAY_INDEX correctly
- Retry failed children individually or resubmit array job

SHOULD:
- Submit array job: `aws batch submit-job --job-name <name> --job-queue <queue> --job-definition <def> --array-properties size=100`
- Configure dependencies: `--depends-on jobId=<id>,type=SEQUENTIAL`

MAY:
- Implement per-index error handling
- Use array job dependencies for pipeline workflows

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix child job failures based on specific error | GREEN | Per-child fix — non-destructive |
| Ensure container uses AWS_BATCH_JOB_ARRAY_INDEX | GREEN | Code fix — applied on next run |
| Retry failed children individually | GREEN | Resubmission — non-destructive |
| Resubmit entire array job | YELLOW | Reruns all children — verify idempotency |
| Configure dependencies between array jobs | GREEN | Configuration — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Array job failure affects a large-scale data processing pipeline
- Partial array completion causes data inconsistency
- Array job dependencies block downstream processing

## Data Sensitivity

- **Classification: MEDIUM**
- Array job indices may map to specific data partitions or customers
- Child job logs may contain partition-specific sensitive data
- Array size and failure patterns reveal data processing architecture
- Dependency configuration reveals pipeline workflow

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest resubmitting an entire array job without verifying idempotency
- **NEVER** suggest ignoring partial array failures without checking data consistency
- **NEVER** suggest exceeding array size limits (10,000) by splitting into uncoordinated jobs

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Retried failed children | Cancel retried children if fix is incorrect |
| Resubmitted entire array job | Cancel array job if resubmission was not needed |
| Changed dependency configuration | Revert dependency settings |

## Output Format

```yaml
root_cause: "array_job_issue — <specific_cause>"
evidence:
  - type: array_status
    content: "<parent and child job status>"
  - type: failed_children
    content: "<failed child indices and reasons>"
severity: MEDIUM
mitigation:
  immediate: "Fix child job failures and retry"
  long_term: "Implement robust array job error handling"
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
