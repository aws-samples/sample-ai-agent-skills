---
title: "E1 — Snapshot Failures"
description: "Diagnose Neptune snapshot creation and restore failures"
status: active
severity: HIGH
triggers:
  - "snapshot failed"
  - "cannot create snapshot"
  - "snapshot error"
  - "backup failed"
owner: devops-agent
objective: "Identify and resolve Neptune snapshot issues"
context: "Neptune supports automated and manual snapshots. Snapshots are stored in S3 (managed by AWS). Snapshot creation is incremental after the first full snapshot."
---

## Phase 1 — Triage

MUST:
- List snapshots: `aws neptune describe-db-cluster-snapshots --db-cluster-identifier <cluster-id>`
- Check snapshot status: `aws neptune describe-db-cluster-snapshots --db-cluster-snapshot-identifier <snapshot-id>`
- Verify cluster is in available state: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Status'`
- Check for backup retention period: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].BackupRetentionPeriod'`

SHOULD:
- Check CloudTrail for snapshot errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDBClusterSnapshot --max-results 10`
- Verify KMS key for encrypted snapshots: `aws kms describe-key --key-id <key-id>`
- Check for concurrent snapshot operations

MAY:
- Verify snapshot copy permissions for cross-account sharing
- Check service quotas for snapshot limits
- Review automated backup window configuration

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (KMS, cluster state, permissions)
- Retry snapshot creation after resolving the issue
- Verify snapshot completes successfully

SHOULD:
- Configure appropriate backup retention period
- Set backup window during low-traffic periods
- Tag snapshots for lifecycle management

MAY:
- Implement cross-region snapshot copy for DR
- Set up automated snapshot monitoring

## Common Issues

- symptoms: "Snapshot creation fails with KMS error"
  diagnosis: "KMS key is not accessible or has been disabled."
  resolution: "Verify KMS key is enabled and the Neptune service has kms:CreateGrant permission."

- symptoms: "Cannot restore from snapshot"
  diagnosis: "Snapshot is in a different region or KMS key is not available."
  resolution: "Copy snapshot to target region. Ensure KMS key is available in target region."

- symptoms: "Automated backups not occurring"
  diagnosis: "Backup retention period is set to 0 or backup window conflicts."
  resolution: "Set backup retention period to at least 1 day."

## Output Format

```yaml
root_cause: "snapshot — <specific_cause>"
evidence:
  - type: snapshot
    content: "<snapshot status and details>"
  - type: cluster
    content: "<cluster backup configuration>"
severity: HIGH
mitigation:
  immediate: "Fix snapshot issue and retry"
  long_term: "Implement backup monitoring and cross-region copies"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — snapshots contain full database copies
- Service Disruption Risk: LOW — snapshot failures do not affect running clusters
- Compliance Impact: HIGH — backup failures affect disaster recovery compliance

## Escalation Conditions
- Automated snapshots fail for multiple consecutive days
- Manual snapshot creation fails with permission errors
- Snapshot copy to another region fails
- Snapshot sharing exposes data to unauthorized accounts
- Snapshot retention policy deletes critical backups

## Data Sensitivity
- Snapshot identifiers and ARNs — INTERNAL
- Snapshot contents — CONFIDENTIAL (full database copy)
- KMS encryption keys for snapshots — CONFIDENTIAL
- Cross-account sharing configurations — CONFIDENTIAL
- Never share snapshots with untrusted AWS accounts

## Prohibited Actions
- Do NOT share snapshots publicly or with unauthorized accounts
- Do NOT delete the only snapshot of a production database
- Do NOT disable automated snapshots on production clusters
- Do NOT copy snapshots to regions without appropriate compliance controls
- Do NOT use unencrypted snapshots for production data
- Do NOT ignore snapshot failure notifications

## Phase 3 — Rollback
- Re-enable automated snapshots if accidentally disabled
- Restore snapshot retention settings to previous values
- Revoke snapshot sharing with unauthorized accounts
- Re-create manual snapshots if critical ones were deleted
- Verify snapshot encryption settings after any changes
- Document snapshot management changes and their rationale

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
