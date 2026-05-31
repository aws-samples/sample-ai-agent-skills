---
title: "F2 — Version Management"
description: "Diagnose S3 versioning management issues including listing, deleting, and cost control"
status: active
severity: MEDIUM
triggers:
  - "Too many versions"
  - "Version storage costs"
  - "List versions"
  - "Delete specific version"
  - "Suspend versioning"
owner: devops-agent
objective: "Help manage S3 object versions effectively and control storage costs"
context: "Versioning cannot be disabled once enabled — only suspended. Suspended versioning stops creating new versions but retains existing ones. Each version is a full copy (not a diff). Storage costs accumulate with every version. Use lifecycle rules to manage version retention."
---

## Phase 1 — Triage

MUST:
- Check versioning status: `aws s3api get-bucket-versioning --bucket <bucket>`
- List versions for a key: `aws s3api list-object-versions --bucket <bucket> --prefix <key>`
- Check lifecycle rules for NoncurrentVersionExpiration: `aws s3api get-bucket-lifecycle-configuration --bucket <bucket>`
- Check storage metrics: `aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name BucketSizeBytes --dimensions Name=BucketName,Value=<bucket> Name=StorageType,Value=StandardStorage --start-time <start> --end-time <end> --period 86400 --statistics Average`

SHOULD:
- Compare current version count vs noncurrent version count
- Estimate storage cost from noncurrent versions
- Check if NewerNoncurrentVersions is configured in lifecycle rules

MAY:
- Use S3 Inventory with version information to audit all versions
- Use S3 Storage Lens for version-related metrics

## Phase 2 — Remediate

MUST:
- Add NoncurrentVersionExpiration lifecycle rule to limit version retention
- Use NewerNoncurrentVersions to keep N most recent versions
- To delete a specific version: `aws s3api delete-object --bucket <bucket> --key <key> --version-id <version-id>`

SHOULD:
- Transition noncurrent versions to cheaper storage before expiring
- Set up S3 Inventory to track version growth over time
- Consider suspending versioning if it is no longer needed (existing versions remain)

MAY:
- Use S3 Batch Operations for bulk version deletion
- Implement a version retention policy based on business requirements

## Common Issues

- symptoms: "Storage costs growing rapidly despite stable data volume"
  diagnosis: "Noncurrent versions accumulating without lifecycle cleanup."
  resolution: "Add NoncurrentVersionExpiration with appropriate NoncurrentDays."

- symptoms: "Cannot disable versioning"
  diagnosis: "Versioning can only be suspended, not disabled. Existing versions remain."
  resolution: "Suspend versioning and add lifecycle rules to expire noncurrent versions."

- symptoms: "Deleting objects does not free storage"
  diagnosis: "DELETE creates delete markers. Noncurrent versions still consume storage."
  resolution: "Delete specific version IDs to permanently remove, or use lifecycle rules."

## Output Format

```yaml
root_cause: "version_management — <specific_cause>"
evidence:
  - type: versioning_status
    content: "<versioning state>"
  - type: version_count
    content: "<current and noncurrent version counts>"
severity: MEDIUM
mitigation:
  immediate: "Add lifecycle rules for version management"
  long_term: "Implement version retention policy with NoncurrentVersionExpiration"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying lifecycle rules and deleting specific versions via put-bucket-lifecycle-configuration and delete-object with version-id. Suspending versioning is state-changing but recoverable. Uses get-bucket-versioning and list-object-versions for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Versioning state changes affect compliance or data protection requirements

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Lifecycle rules reveal data retention strategy"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

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
