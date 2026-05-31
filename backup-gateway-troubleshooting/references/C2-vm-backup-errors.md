---
title: "C2 — VM Backup Errors"
description: "Diagnose virtual machine backup failures"
status: active
severity: CRITICAL
triggers: ["backup failed", "VM backup error", "backup job failed", "snapshot error"]
owner: devops-agent
objective: "Resolve VM backup failures in Backup Gateway"
context: "Backups use VMware snapshots and Changed Block Tracking (CBT). Failures can be gateway-side, hypervisor-side, or AWS-side."
---

## Phase 1 — Triage

MUST:
- Check failed backup jobs: `aws backup list-backup-jobs --by-resource-type VIRTUALMACHINE --by-state FAILED --max-results 10`
- Get backup job details: `aws backup describe-backup-job --backup-job-id <job-id>`
- Verify gateway is online: `aws backup-gateway list-gateways --query 'Gateways[*].{Name:GatewayDisplayName,LastSeen:LastSeenTime}'`

SHOULD:
- Check if CBT is enabled on the VM in vCenter
- Verify the VM does not have existing snapshots that block new ones
- Check gateway resource utilization (CPU, memory, disk)

MAY:
- Review CloudWatch metrics for backup throughput: `aws cloudwatch get-metric-statistics --namespace AWS/BackupGateway --metric-name GatewayBytesUploaded --dimensions Name=GatewayArn,Value=<arn> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check backup vault policies: `aws backup get-backup-vault-access-policy --backup-vault-name <vault>`

## Phase 2 — Remediate

MUST:
- Fix the specific failure identified in the backup job details
- Remove stale VMware snapshots blocking new snapshot creation
- Ensure CBT is enabled for incremental backups

SHOULD:
- Verify VMware Tools is installed for application-consistent backups
- Check backup vault encryption and access policies

MAY:
- Adjust backup window to avoid peak vCenter activity
- Increase gateway VM resources if performance is the bottleneck

## Safety Ratings

```
safety_ratings:
  - "backup list-backup-jobs: GREEN — read-only job listing"
  - "backup describe-backup-job: GREEN — read-only job details"
  - "backup-gateway list-gateways: GREEN — read-only gateway listing"
  - "VMware snapshot removal: YELLOW — modifies VM snapshot chain"
```

## Escalation Conditions

- Backup failures affect compliance or SLA requirements
- VM has a corrupted snapshot chain that cannot be resolved
- Gateway resources are insufficient and require VM reconfiguration
- Backup vault policies block backup storage

## Data Sensitivity

- `aws backup list-backup-jobs`: MEDIUM — reveals backup job history and resource ARNs
- `aws backup describe-backup-job`: MEDIUM — reveals backup details and error messages
- `aws backup get-backup-vault-access-policy`: HIGH — reveals vault access control
- VMware snapshot data: HIGH — contains VM disk data

## Prohibited Actions

- NEVER delete VMware snapshots without understanding the snapshot chain
- NEVER disable CBT to work around backup errors — this forces full backups
- NEVER modify backup vault policies without security team approval
- NEVER cancel running backup jobs without understanding the impact on the backup chain

## Phase 3 — Rollback

1. If a VMware snapshot was created during troubleshooting, remove it to free disk space
2. If CBT was reset, plan for the next backup to be a full backup
3. Verify the backup schedule is intact after troubleshooting
4. Check that the backup vault has sufficient capacity
5. Document the failure cause and resolution for the operations team

## Common Issues

- symptoms: "Backup job fails with 'Snapshot creation failed'"
  diagnosis: "VM has too many existing snapshots or CBT is disabled."
  resolution: "Remove stale snapshots in vCenter. Enable CBT on the VM. Retry the backup."

## Output Format

```yaml
root_cause: "vm_backup — <specific_cause>"
evidence:
  - type: backup_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix backup failure"
  long_term: "Monitor snapshot health and CBT status"
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
