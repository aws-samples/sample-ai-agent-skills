---
title: "I1 — Snapshot Failures"
description: "Diagnose and resolve Redshift snapshot creation failures"
status: active
severity: HIGH
triggers:
  - "snapshot failed"
  - "backup failed"
  - "snapshot creation"
  - "automated snapshot"
owner: devops-agent
objective: "Identify and fix snapshot creation failures"
context: "Redshift creates automated snapshots every 8 hours or 5 GB of changes. Manual snapshots are user-initiated. Snapshots are stored in S3 (managed by Redshift). Cross-region snapshot copy enables DR. Snapshot failures can block cluster operations."
---

## Phase 1 — Triage

MUST:
- Check snapshot status: `aws redshift describe-cluster-snapshots --cluster-identifier <cluster-id> --query 'Snapshots[*].{id:SnapshotIdentifier,status:Status,type:SnapshotType,time:SnapshotCreateTime}' --max-records 10`
- Check cluster events for snapshot errors: `aws redshift describe-events --source-identifier <cluster-id> --source-type cluster --duration 1440 --query "Events[?contains(Message, 'snapshot')]"`
- Check snapshot schedule: `aws redshift describe-snapshot-schedules --cluster-identifier <cluster-id>`
- Verify cluster status is available: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].ClusterStatus'`

SHOULD:
- Check automated snapshot retention period
- Verify cross-region copy configuration if enabled
- Check for concurrent operations blocking snapshots

MAY:
- Check snapshot quota limits
- Review CloudTrail for snapshot API errors
- Check KMS key status if cluster is encrypted

## Phase 2 — Remediate

MUST:
- If cluster not available: resolve cluster issue first — snapshots require available cluster
- If KMS key issue: fix KMS key permissions or re-enable key
- If quota exceeded: delete old manual snapshots or request quota increase
- If cross-region copy failing: check destination region KMS key and permissions

SHOULD:
- Configure automated snapshot retention (1-35 days)
- Set up snapshot schedule for consistent backup timing
- Enable cross-region snapshot copy for DR

MAY:
- Implement snapshot lifecycle management
- Set up CloudWatch alarms for snapshot failures
- Tag snapshots for cost tracking

## Common Issues

- symptoms: "Automated snapshots not being created"
  diagnosis: "Automated snapshot retention set to 0 (disabled) or cluster in modifying state."
  resolution: "Set retention period > 0. Ensure cluster is in available state."

- symptoms: "Cross-region snapshot copy fails"
  diagnosis: "Destination region KMS key not configured or IAM permissions missing."
  resolution: "Configure KMS key in destination region. Grant cross-region copy permissions."

## Output Format

```yaml
root_cause: "Snapshot failure — <specific_cause>"
evidence:
  - type: snapshot_status
    content: "<describe-cluster-snapshots output>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry snapshot"
  long_term: "Configure automated snapshots, enable cross-region copy, monitor snapshot health"
```


## Safety Ratings
```
safety_ratings:
  - "Check snapshot status: GREEN — read-only API call"
  - "Check cluster events: GREEN — read-only API call"
  - "Check snapshot schedule: GREEN — read-only API call"
  - "Delete old manual snapshots: YELLOW — permanently removes backup point"
  - "Configure automated snapshot retention: GREEN — sets retention policy"
  - "Enable cross-region copy: YELLOW — copies snapshots to another region with cost implications"
  - "Fix KMS key permissions: YELLOW — changes encryption key access"
```

## Escalation Conditions
- Cluster serves production analytics
- Automated snapshots not being created (backup gap)
- Cross-region snapshot copy failing (DR gap)
- KMS key issues blocking snapshot creation
- Snapshot quota exceeded requiring cleanup

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Snapshot identifiers: backup point references"
    - "Snapshot size: data volume information"
    - "KMS key configuration: encryption details"
    - "Cross-region copy settings: DR architecture"
  handling: "Snapshots contain full cluster data. Ensure snapshot access is restricted to authorized personnel."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER delete the most recent snapshot without confirming newer backups exist
- NEVER set automated snapshot retention to 0 (disables automated backups)
- NEVER disable cross-region copy without confirming DR requirements are met elsewhere

## Phase 3 — Rollback
- If manual snapshots were deleted: CANNOT be recovered — ensure they were truly not needed
- If automated snapshot retention was changed: revert to previous retention period
- If cross-region copy was enabled: disable cross-region copy and delete destination snapshots if not needed
- If snapshot schedule was modified: restore previous schedule configuration

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
