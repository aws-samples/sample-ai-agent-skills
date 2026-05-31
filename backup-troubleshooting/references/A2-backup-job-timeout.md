---
title: "A2 — Backup Job Timeout"
description: "Diagnose why AWS Backup jobs are timing out or taking too long"
status: active
severity: HIGH
triggers:
  - "backup timeout"
  - "backup job expired"
  - "completion window"
  - "backup taking too long"
  - "backup job cancelled"
owner: devops-agent
objective: "Identify why backup jobs exceed their completion window and resolve timeout issues"
context: "Backup jobs have a start window (when the job can begin) and a completion window (maximum duration). If a job does not finish within the completion window, it is cancelled. Large resources, throttling, or resource contention can cause timeouts."
---

## Phase 1 — Triage

MUST:
- Describe the timed-out job: `aws backup describe-backup-job --backup-job-id <job-id>`
- Check the StatusMessage for timeout-specific errors
- Check the CreationDate, CompletionDate, and BackupSizeInBytes fields
- Identify the resource type and size being backed up
- Check the backup plan's completion window: `aws backup get-backup-plan --backup-plan-id <plan-id>`

SHOULD:
- Check if the resource is unusually large (e.g., large EBS volume, large RDS instance)
- Check for concurrent backup jobs that may cause throttling: `aws backup list-backup-jobs --by-state RUNNING`
- Check CloudWatch metrics for the resource (CPU, I/O) during the backup window
- Verify no service-level throttling is occurring

MAY:
- Check if EBS snapshots are incremental (first snapshot is always full and takes longer)
- Review historical backup job durations for the same resource
- Check for resource-level I/O contention during backup

## Phase 2 — Remediate

MUST:
- Increase the completion window if the resource legitimately requires more time
- Adjust the start window to begin during low-activity periods
- Retry the backup after addressing the root cause

SHOULD:
- Stagger backup schedules to avoid concurrent large backups
- For large EBS volumes, ensure the first full snapshot completes before relying on incremental
- Consider using a wider backup window for large resources

MAY:
- Split large resources into smaller components where possible
- Enable EBS fast snapshot restore for frequently backed-up volumes
- Monitor backup duration trends to proactively adjust windows

## Common Issues

- symptoms: "Backup job status is EXPIRED"
  diagnosis: "The job did not complete within the completion window."
  resolution: "Increase the completion window in the backup plan rule. Default is 7 days."

- symptoms: "First backup of a large EBS volume times out"
  diagnosis: "The initial EBS snapshot is a full copy and takes significantly longer than incremental."
  resolution: "Set a longer completion window for the first backup. Subsequent backups will be incremental."

- symptoms: "Multiple backup jobs timing out simultaneously"
  diagnosis: "Too many concurrent backup jobs causing API throttling or I/O contention."
  resolution: "Stagger backup schedules across different start windows."

## Output Format

```yaml
root_cause: "backup_timeout — <specific_cause>"
evidence:
  - type: backup_job
    content: "<job details with timing information>"
  - type: backup_plan
    content: "<completion window configuration>"
  - type: resource_size
    content: "<resource size and type>"
severity: HIGH
mitigation:
  immediate: "Adjust completion window and retry the backup"
  long_term: "Optimize backup schedules and monitor duration trends"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase completion window | YELLOW | Changes backup plan timing — may affect scheduling |
| Adjust start window to low-activity periods | YELLOW | Changes when backups run |
| Retry backup after fixing root cause | GREEN | Re-executes backup — non-destructive |
| Stagger backup schedules | YELLOW | Changes multiple backup plan schedules |
| Enable EBS fast snapshot restore | YELLOW | Incurs additional cost |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Completion window changes affect backup plan compliance with retention policies
- Schedule changes may create gaps in backup coverage

## Data Sensitivity

- Backup job timing reveals operational patterns and maintenance windows
- Resource sizes reveal data volume information
- Backup plan configuration reveals data protection strategy

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER reduce completion window below what the resource needs to complete backup
- NEVER remove backup schedules to fix timeout issues

## Phase 3 — Rollback

- If completion window was changed: restore original window in backup plan
- If start window was adjusted: restore original schedule
- If backup schedules were staggered: restore original schedule timing
- Document all backup plan changes for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
