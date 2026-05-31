---
title: "D1 — Restore Failures"
description: "Diagnose VM restore failures"
status: active
severity: CRITICAL
triggers: ["restore failed", "cannot restore VM", "restore job error", "recovery failed"]
owner: devops-agent
objective: "Resolve VM restore failures in Backup Gateway"
context: "Restore creates a NEW VM in vCenter. It does NOT overwrite the original. Requires sufficient resources on the target ESXi host."
---

## Phase 1 — Triage

MUST:
- Check failed restore jobs: `aws backup list-restore-jobs --by-status FAILED --max-results 10`
- Get restore job details: `aws backup describe-restore-job --restore-job-id <job-id>`
- Verify gateway is online and connected to the target hypervisor

SHOULD:
- Check target ESXi host has sufficient resources (CPU, memory, storage)
- Verify the target datastore has enough free space

MAY:
- Review CloudWatch metrics for restore throughput
- Check if the backup recovery point is valid: `aws backup describe-recovery-point --backup-vault-name <vault> --recovery-point-arn <arn>`

## Phase 2 — Remediate

MUST:
- Fix the specific failure identified in the restore job details
- Ensure target ESXi host has sufficient resources
- Verify the target datastore has enough free space for the restored VM

SHOULD:
- Select an appropriate target host and datastore before retrying
- Verify network connectivity between gateway and target ESXi host

MAY:
- Restore to a different ESXi host if the original target lacks resources
- Use a different datastore if storage is the constraint

## Safety Ratings

```
safety_ratings:
  - "backup list-restore-jobs: GREEN — read-only job listing"
  - "backup describe-restore-job: GREEN — read-only job details"
  - "backup describe-recovery-point: GREEN — read-only recovery point details"
  - "backup start-restore-job: YELLOW — creates new VM in vCenter"
```

## Escalation Conditions

- Restore failure affects disaster recovery SLA
- Recovery point is corrupted or inaccessible
- Target environment lacks resources for the restored VM
- Multiple restore attempts fail with different errors

## Data Sensitivity

- `aws backup list-restore-jobs`: MEDIUM — reveals restore history
- `aws backup describe-restore-job`: MEDIUM — reveals restore details and target configuration
- `aws backup describe-recovery-point`: MEDIUM — reveals backup metadata
- Restored VM data: HIGH — contains production data from the backup

## Prohibited Actions

- NEVER assume restore overwrites the original VM — it creates a new one
- NEVER restore to production without verifying the target environment
- NEVER delete recovery points to troubleshoot restore failures
- NEVER skip post-restore network configuration on the new VM

## Phase 3 — Rollback

1. If a partial restore created a VM, clean it up in vCenter if not needed
2. Verify the recovery point is still valid after failed restore attempts
3. Check that the failed restore did not consume excessive storage on the target datastore
4. Document the restore failure and target environment requirements
5. Verify the backup schedule is unaffected by restore operations

## Common Issues

- symptoms: "Restore job fails with 'Insufficient resources on target host'"
  diagnosis: "Target ESXi host does not have enough CPU, memory, or storage for the restored VM."
  resolution: "Select a different target host with sufficient resources. Verify datastore free space exceeds the VM disk size."

## Output Format

```yaml
root_cause: "restore_failure — <specific_cause>"
evidence:
  - type: restore_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix restore prerequisites"
  long_term: "Pre-validate target environment capacity"
```

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
