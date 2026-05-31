---
title: "D2 — Cross-Account Replication"
description: "Diagnose cross-account S3 replication failures"
status: active
severity: HIGH
triggers:
  - "Cross-account replication"
  - "Replication to another account"
  - "Destination bucket policy replication"
owner: devops-agent
objective: "Configure cross-account replication with correct permissions on both sides"
context: "Cross-account replication requires the destination bucket policy to allow the source account's replication role. If the destination uses KMS encryption, the source replication role needs access to the destination KMS key. Object ownership should be BucketOwnerEnforced on the destination."
---

## Phase 1 — Triage

MUST:
- Check replication config in source account: `aws s3api get-bucket-replication --bucket <source-bucket>`
- Check destination bucket policy for replication role access: `aws s3api get-bucket-policy --bucket <dest-bucket>`
- Verify the replication role's trust policy allows sts:AssumeRole from s3.amazonaws.com
- Check destination bucket ownership controls: `aws s3api get-bucket-ownership-controls --bucket <dest-bucket>`

SHOULD:
- Verify KMS key access if destination uses SSE-KMS
- Check that the replication rule specifies the destination account ID
- Verify AccessControlTranslation is set to Destination if using ownership override

MAY:
- Check CloudTrail in both accounts for replication-related events
- Verify SCPs in both accounts allow the required actions

## Phase 2 — Remediate

MUST:
- Add destination bucket policy allowing the replication role:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<source-account>:role/<replication-role>"},
    "Action": ["s3:ReplicateObject", "s3:ReplicateDelete", "s3:ReplicateTags", "s3:ObjectOwnerOverrideToBucketOwner"],
    "Resource": "arn:aws:s3:::<dest-bucket>/*"
  }
  ```
- Set destination bucket ownership to BucketOwnerEnforced
- Include Account and AccessControlTranslation in the replication rule

SHOULD:
- If destination uses KMS: grant the replication role kms:Encrypt on the destination key
- Add s3:GetBucketVersioning on the destination bucket to the replication role
- Test with a new object upload and verify it appears in the destination

MAY:
- Enable S3 Replication Time Control for SLA-backed cross-account replication
- Use S3 Batch Replication for existing objects

## Common Issues

- symptoms: "Replication fails with AccessDenied on destination"
  diagnosis: "Destination bucket policy does not allow the source replication role."
  resolution: "Add the replication role to the destination bucket policy with ReplicateObject permissions."

- symptoms: "Objects replicate but destination account cannot access them"
  diagnosis: "Object ownership is not set to BucketOwnerEnforced on destination."
  resolution: "Set BucketOwnerEnforced on destination and add ObjectOwnerOverrideToBucketOwner to the replication rule."

- symptoms: "KMS-encrypted objects fail to replicate cross-account"
  diagnosis: "Replication role lacks kms:Encrypt on the destination KMS key."
  resolution: "Grant the replication role kms:Encrypt on the destination key and kms:Decrypt on the source key."

## Output Format

```yaml
root_cause: "cross_account_replication — <specific_cause>"
evidence:
  - type: replication_config
    content: "<replication rule>"
  - type: dest_bucket_policy
    content: "<destination bucket policy>"
severity: HIGH
mitigation:
  immediate: "Fix destination bucket policy and ownership settings"
  long_term: "Document cross-account replication requirements and monitor with RTC"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying destination bucket policies to allow cross-account replication roles. Requires put-bucket-policy on the destination and ownership control changes. Cross-account policy changes carry elevated risk. |

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
