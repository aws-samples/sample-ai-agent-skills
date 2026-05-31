---
title: "G1 — Snapshot Failures"
description: "Diagnose DocumentDB snapshot creation and management failures"
status: active
severity: HIGH
triggers:
  - "snapshot failed"
  - "snapshot error"
  - "cannot create snapshot"
  - "snapshot timeout"
  - "backup failed"
owner: devops-agent
objective: "Identify and resolve snapshot creation and management failures"
context: "DocumentDB supports automatic daily snapshots and manual snapshots. Snapshot failures can be caused by KMS key issues, service quotas, cluster state, or concurrent operations. Snapshots are cluster-level, not collection-level."
---

## Phase 1 — Triage

MUST:
- List cluster snapshots: `aws docdb describe-db-cluster-snapshots --db-cluster-identifier <cluster-id>`
- Check snapshot status for failures
- Verify cluster is in available state: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check KMS key status: `aws kms describe-key --key-id <key-id>`

SHOULD:
- Check cluster events for snapshot errors: `aws docdb describe-events --source-type db-cluster --duration 1440`
- Verify automatic backup configuration (retention period, backup window)
- Check service quotas for snapshot count
- Review CloudTrail for snapshot API errors

MAY:
- Check for concurrent snapshot operations
- Verify snapshot copy operations if cross-region
- Monitor snapshot creation duration

## Phase 2 — Remediate

MUST:
- Fix KMS key access if that is the cause
- Ensure cluster is in available state before creating snapshots
- Verify automatic backup retention is > 0

SHOULD:
- Set backup window during low-traffic periods
- Monitor snapshot creation success with CloudWatch
- Maintain appropriate snapshot retention

MAY:
- Automate snapshot management with AWS Backup
- Set up alerts for snapshot failures

## Common Issues

- symptoms: "Automatic backup not being created"
  diagnosis: "Backup retention period is set to 0 (disabled) or backup window conflicts."
  resolution: "Set BackupRetentionPeriod > 0. Verify PreferredBackupWindow is configured."

- symptoms: "Manual snapshot fails with quota error"
  diagnosis: "Maximum number of manual snapshots reached."
  resolution: "Delete old manual snapshots. Check quota with aws service-quotas."

- symptoms: "Snapshot creation takes very long"
  diagnosis: "Large cluster storage or heavy write activity during snapshot."
  resolution: "Snapshots are incremental — first snapshot takes longest. Schedule during low-write periods."

## Output Format

```yaml
root_cause: "snapshot_failure — <specific_cause>"
evidence:
  - type: snapshots
    content: "<snapshot list and status>"
  - type: cluster
    content: "<cluster status and backup configuration>"
  - type: kms
    content: "<KMS key status>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry snapshot"
  long_term: "Configure automatic backups and monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Snapshots contain complete database data |
| Blast Radius | MEDIUM — Snapshot failures affect backup and recovery capability |
| Rollback Complexity | LOW — Fix configuration and retry snapshot |
| Compliance Scope | CRITICAL — Backups are a mandatory compliance control |

## Escalation Conditions

- Escalate IMMEDIATELY to operations if automatic backups are failing
- Escalate to security if KMS key issues prevent snapshot creation
- Escalate to compliance if backup gaps exceed retention requirements

## Data Sensitivity

- **Snapshots** contain complete database data — apply same security as the cluster
- **KMS key** used for snapshot encryption must remain accessible
- **Cross-region snapshot copies** must comply with data residency requirements

## PROHIBITED Actions

- Do NOT set backup retention to 0 on production clusters
- Do NOT delete manual snapshots without confirming they are no longer needed
- Do NOT copy snapshots to regions that violate data residency requirements
- Do NOT share snapshots with unauthorized accounts

## Phase 3 — Rollback

- Restore backup retention if it was changed: `aws docdb modify-db-cluster --db-cluster-identifier <id> --backup-retention-period <days>`
- Fix KMS key access if that caused the failure
- Manually create a snapshot to verify the fix: `aws docdb create-db-cluster-snapshot --db-cluster-identifier <id> --db-cluster-snapshot-identifier <name>`

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
