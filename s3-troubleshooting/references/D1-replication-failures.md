---
title: "D1 — Replication Failures"
description: "Diagnose S3 replication failures including versioning, IAM role, and destination issues"
status: active
severity: HIGH
triggers:
  - "Replication not working"
  - "Objects not replicating"
  - "Replication failed"
  - "ReplicationConfiguration"
owner: devops-agent
objective: "Identify why S3 replication is failing and fix the configuration"
context: "S3 replication requires versioning enabled on BOTH source and destination buckets. It requires an IAM role with permissions to read from source and write to destination. Replication only applies to NEW objects written after the rule is created — use S3 Batch Replication for existing objects. Delete markers are optionally replicated."
---

## Phase 1 — Triage

MUST:
- Check replication configuration: `aws s3api get-bucket-replication --bucket <source-bucket>`
- Verify versioning on source: `aws s3api get-bucket-versioning --bucket <source-bucket>`
- Verify versioning on destination: `aws s3api get-bucket-versioning --bucket <dest-bucket>`
- Check the replication IAM role permissions: `aws iam get-role-policy --role-name <replication-role> --policy-name <policy>`
- Check replication metrics: `aws s3api get-bucket-replication --bucket <source-bucket> --query 'ReplicationConfiguration.Rules[].ExistingObjectReplication'`

SHOULD:
- Check S3 replication metrics in CloudWatch: ReplicationLatency, OperationsPendingReplication
- Verify the destination bucket policy allows the replication role
- Check if the source objects are encrypted with KMS (requires additional KMS permissions)

MAY:
- Check CloudTrail for failed replication events
- Verify no SCPs block the replication role's actions

## Phase 2 — Remediate

MUST:
- Enable versioning on both source and destination if not already enabled
- Fix the IAM role to include: s3:GetReplicationConfiguration, s3:ListBucket on source; s3:GetObjectVersionForReplication, s3:GetObjectVersionAcl on source objects; s3:ReplicateObject, s3:ReplicateDelete, s3:ReplicateTags on destination
- For KMS-encrypted objects: add kms:Decrypt on source key and kms:Encrypt on destination key

SHOULD:
- Enable S3 Replication Metrics and Notifications for monitoring
- Use S3 Batch Replication for existing objects that were not replicated
- Enable delete marker replication if needed

MAY:
- Enable S3 Replication Time Control (RTC) for SLA-backed replication
- Set up CloudWatch alarms on ReplicationLatency and OperationsPendingReplication

## Common Issues

- symptoms: "Replication enabled but no objects are replicating"
  diagnosis: "Replication only applies to new objects. Existing objects are not replicated."
  resolution: "Use S3 Batch Replication to replicate existing objects."

- symptoms: "Some objects replicate but KMS-encrypted objects do not"
  diagnosis: "Replication role lacks KMS permissions for the encryption key."
  resolution: "Add kms:Decrypt for the source key and kms:Encrypt for the destination key to the replication role."

- symptoms: "Versioning was suspended on destination"
  diagnosis: "Replication requires versioning enabled on both buckets. Suspended versioning blocks replication."
  resolution: "Re-enable versioning on the destination bucket."

## Output Format

```yaml
root_cause: "replication_failure — <specific_cause>"
evidence:
  - type: replication_config
    content: "<replication configuration>"
  - type: versioning_status
    content: "<source and destination versioning>"
severity: HIGH
mitigation:
  immediate: "Fix the replication configuration issue"
  long_term: "Enable replication metrics and CloudWatch alarms"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses get-bucket-replication, get-bucket-versioning, get-role-policy. Remediation fixes IAM role permissions and replication configuration, not direct bucket access controls. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Bucket contains sensitive/regulated data (PII, PHI, financial)
- Cross-account access changes are needed
- Encryption configuration changes affect multiple consumers

## Rollback
1. Before any bucket policy change: Save current policy with `aws s3api get-bucket-policy`
2. Before ACL changes: Save current ACL with `aws s3api get-bucket-acl`
3. After change: Verify access works without granting excessive permissions
4. If change causes issues: Restore the saved policy/ACL immediately
5. Cleanup: Remove any temporary access grants

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-bucket-policy` | HIGH | Contains access rules — redact principals |
| `get-bucket-acl` | MEDIUM | Shows grantees — summarize |
| `get-public-access-block` | MEDIUM | Security posture — safe to include |
| `list-objects` | LOW | Object keys only — safe to include |

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` in bucket policy without Condition keys
- NEVER suggest removing bucket encryption to fix access issues
- NEVER suggest making a bucket public to resolve CORS or access issues
- NEVER suggest `s3:*` in any IAM or bucket policy fix
- ALWAYS use least-privilege: grant only the specific S3 action needed
- ALWAYS check both account-level AND bucket-level Block Public Access

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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
