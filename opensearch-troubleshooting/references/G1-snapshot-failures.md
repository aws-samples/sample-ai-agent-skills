---
title: "G1 — OpenSearch Snapshot Failures"
description: "Diagnose and resolve manual and automated snapshot creation failures"
status: active
severity: HIGH
triggers:
  - "snapshot failed"
  - "snapshot error"
  - "backup failed"
  - "AutomatedSnapshotFailure"
  - "snapshot in progress"
owner: devops-agent
objective: "Identify the cause of snapshot failures and restore backup capability"
context: "OpenSearch supports automated snapshots (daily, AWS-managed, stored internally) and manual snapshots (user-initiated, stored in S3). Automated snapshots cannot be restored to a different domain. Manual snapshots require registering an S3 repository with an IAM role. Snapshot failures are commonly caused by RED cluster health, S3 permissions, KMS key access, or concurrent snapshot operations. Only one snapshot can run at a time."
---

## Phase 1 — Triage

MUST:
- Check automated snapshot failure metric: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name AutomatedSnapshotFailure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check snapshot status: `curl -XGET "https://<endpoint>/_snapshot/_status?pretty"`
- List snapshot repositories: `curl -XGET "https://<endpoint>/_snapshot?pretty"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Check recent snapshots: `curl -XGET "https://<endpoint>/_snapshot/<repo>/_all?pretty"`

SHOULD:
- Check snapshot repository configuration: `curl -XGET "https://<endpoint>/_snapshot/<repo>?pretty"`
- Check for concurrent snapshot operations: `curl -XGET "https://<endpoint>/_snapshot/_status?pretty"`
- Check domain snapshot config: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.SnapshotOptions'`

MAY:
- Check S3 bucket permissions for manual snapshot repository
- Check KMS key policy if using encrypted snapshots
- Check CloudTrail for snapshot-related API calls

## Phase 2 — Remediate

MUST:
- If RED cluster: resolve unassigned primaries first (see A1)
- If S3 permission error: verify IAM role has s3:ListBucket, s3:GetObject, s3:PutObject on the bucket
- If concurrent snapshot: wait for current snapshot to complete before starting another
- If repository not registered: register it (see G3)

SHOULD:
- Set up CloudWatch alarm on AutomatedSnapshotFailure
- Test manual snapshot after fixing: `curl -XPUT "https://<endpoint>/_snapshot/<repo>/test-snapshot?pretty"`
- Verify snapshot completion: `curl -XGET "https://<endpoint>/_snapshot/<repo>/test-snapshot?pretty"`

MAY:
- Delete old snapshots to free S3 space: `curl -XDELETE "https://<endpoint>/_snapshot/<repo>/<old-snapshot>"`
- Schedule manual snapshots using Lambda or EventBridge

## Common Issues

- symptoms: "AutomatedSnapshotFailure metric is 1"
  diagnosis: "Cluster health is RED. Automated snapshots fail when primaries are unassigned."
  resolution: "Fix RED cluster health first. Automated snapshots will resume."

- symptoms: "Manual snapshot fails with repository_exception"
  diagnosis: "S3 repository not registered or IAM role lacks permissions."
  resolution: "Register repository with correct IAM role and S3 bucket. See G3."

- symptoms: "Snapshot stuck in IN_PROGRESS state"
  diagnosis: "Large index or slow S3 upload. Snapshot taking longer than expected."
  resolution: "Wait for completion. Check _snapshot/_status for progress. Do not start another."

## Output Format

```yaml
root_cause: "snapshot_failure — <specific_cause>"
evidence:
  - type: snapshot_status
    content: "<snapshot status and errors>"
  - type: cluster_health
    content: "<cluster health status>"
  - type: repository_config
    content: "<snapshot repository configuration>"
severity: HIGH
mitigation:
  immediate: "Fix underlying issue (cluster health, permissions) and retry snapshot"
  long_term: "Set up snapshot monitoring alarms, automate manual snapshots"
```


## Safety Ratings
```
safety_ratings:
  - "Check snapshot status and metrics: GREEN — read-only diagnostics"
  - "List repositories and snapshots: GREEN — read-only API calls"
  - "Take manual snapshot: GREEN — creates backup without affecting cluster"
  - "Delete old snapshots: YELLOW — removes backup points"
  - "Fix IAM role permissions: YELLOW — changes S3 access scope"
```

## Escalation Conditions
- Domain serves production search
- Automated snapshots failing (backup gap)
- RED cluster blocking snapshot creation
- S3 repository permissions requiring IAM changes
- Concurrent snapshot blocking new backup

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Snapshot data: full index data in S3"
    - "S3 repository configuration: bucket and IAM role details"
    - "Snapshot metadata: index names and sizes"
  handling: "Snapshots contain full index data. Ensure S3 bucket has appropriate encryption and access controls."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER delete the most recent successful snapshot without confirming newer backups exist
- NEVER start a new snapshot while another is in progress

## Phase 3 — Rollback
- If old snapshots were deleted: CANNOT be recovered
- If IAM role permissions were changed: revert to previous policy
- If repository was re-registered: restore previous repository configuration

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
