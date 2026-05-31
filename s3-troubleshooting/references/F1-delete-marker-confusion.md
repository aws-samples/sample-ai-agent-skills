---
title: "F1 — Delete Marker Confusion"
description: "Diagnose issues caused by S3 delete markers in versioned buckets"
status: active
severity: MEDIUM
triggers:
  - "Delete marker"
  - "Object deleted but still billed"
  - "Cannot find deleted object"
  - "Restore deleted object"
owner: devops-agent
objective: "Explain delete marker behavior and help restore or clean up versioned objects"
context: "In versioned buckets, deleting an object creates a delete marker — a zero-byte placeholder that makes the object appear deleted. Previous versions still exist and incur storage costs. Removing the delete marker restores the most recent version. Delete markers are NOT objects and have no storage cost themselves."
---

## Phase 1 — Triage

MUST:
- Check versioning status: `aws s3api get-bucket-versioning --bucket <bucket>`
- List object versions including delete markers: `aws s3api list-object-versions --bucket <bucket> --prefix <key>`
- Identify delete markers (IsDeleteMarker: true) and their version IDs
- Check if previous versions exist behind the delete marker

SHOULD:
- Count total versions and delete markers for the prefix
- Check if lifecycle rules are managing delete markers (ExpiredObjectDeleteMarker)
- Verify the user understands that "deleted" objects still have versions

MAY:
- Use S3 Inventory to audit delete markers across the bucket
- Check storage costs from retained versions

## Phase 2 — Remediate

MUST:
- To restore: delete the delete marker by version ID: `aws s3api delete-object --bucket <bucket> --key <key> --version-id <delete-marker-version-id>`
- To permanently delete: delete all versions including the delete marker
- To clean up: add ExpiredObjectDeleteMarker lifecycle rule

SHOULD:
- Add NoncurrentVersionExpiration lifecycle rule to limit version accumulation
- Educate users that DELETE on versioned buckets creates markers, not permanent deletes
- Use `--version-id` for permanent deletes of specific versions

MAY:
- Use S3 Batch Operations to clean up delete markers at scale
- Implement a retention policy with lifecycle rules

## Common Issues

- symptoms: "Deleted an object but storage costs did not decrease"
  diagnosis: "Delete created a marker. Previous versions still exist and are billed."
  resolution: "Use NoncurrentVersionExpiration lifecycle rule or delete versions explicitly."

- symptoms: "Object not found but it was there yesterday"
  diagnosis: "A delete marker was created, hiding the object."
  resolution: "List versions to find the delete marker. Delete the marker to restore."

- symptoms: "Thousands of delete markers slowing down list operations"
  diagnosis: "Repeated delete/recreate cycles accumulate delete markers."
  resolution: "Add ExpiredObjectDeleteMarker: true to lifecycle rules."

## Output Format

```yaml
root_cause: "delete_marker — <specific_cause>"
evidence:
  - type: object_versions
    content: "<version listing with delete markers>"
  - type: versioning_status
    content: "<versioning state>"
severity: MEDIUM
mitigation:
  immediate: "Remove delete marker to restore, or delete all versions to permanently remove"
  long_term: "Implement lifecycle rules for version and delete marker management"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves deleting delete markers and object versions via delete-object with version-id. Removing delete markers restores objects; deleting versions is permanent but targeted. Uses list-object-versions and get-bucket-versioning for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Version deletion affects compliance or audit trail requirements

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
