---
title: "E2 — Lifecycle Expiration Issues"
description: "Diagnose lifecycle expiration issues including versioned objects and delete markers"
status: active
severity: MEDIUM
triggers:
  - "Objects not expiring"
  - "Lifecycle expiration not working"
  - "Delete markers accumulating"
  - "Noncurrent versions not deleted"
owner: devops-agent
objective: "Identify why lifecycle expiration rules are not removing objects as expected"
context: "Lifecycle expiration behaves differently for versioned and unversioned buckets. For versioned buckets, expiration creates a delete marker (does not permanently delete). NoncurrentVersionExpiration removes old versions. ExpiredObjectDeleteMarker removes delete markers with no noncurrent versions. Object Lock retention prevents expiration."
---

## Phase 1 — Triage

MUST:
- Check lifecycle configuration: `aws s3api get-bucket-lifecycle-configuration --bucket <bucket>`
- Check versioning status: `aws s3api get-bucket-versioning --bucket <bucket>`
- If versioned, check for NoncurrentVersionExpiration rules
- Check if Object Lock is enabled: `aws s3api get-object-lock-configuration --bucket <bucket>`

SHOULD:
- List object versions to see delete markers: `aws s3api list-object-versions --bucket <bucket> --prefix <prefix> --max-keys 10`
- Check if ExpiredObjectDeleteMarker cleanup is configured
- Verify the rule filter matches the target objects

MAY:
- Use S3 Inventory to count noncurrent versions and delete markers
- Check storage costs from noncurrent versions

## Phase 2 — Remediate

MUST:
- For versioned buckets: add NoncurrentVersionExpiration to remove old versions
- Add ExpiredObjectDeleteMarker: true to clean up orphaned delete markers
- If Object Lock is blocking: wait for retention to expire or use Governance mode bypass

SHOULD:
- Set NoncurrentDays to control how long old versions are kept
- Set NewerNoncurrentVersions to keep a specific number of recent versions
- Combine expiration with transition rules (transition old versions to Glacier before expiring)

MAY:
- Use S3 Batch Operations to delete specific versions in bulk
- Set up S3 Storage Lens to monitor version accumulation

## Common Issues

- symptoms: "Expiration rule set but objects still exist"
  diagnosis: "Bucket has versioning enabled. Expiration creates delete markers, not permanent deletes."
  resolution: "Add NoncurrentVersionExpiration to permanently remove old versions."

- symptoms: "Delete markers accumulating and increasing list-objects latency"
  diagnosis: "No ExpiredObjectDeleteMarker cleanup rule configured."
  resolution: "Add ExpiredObjectDeleteMarker: true to the lifecycle rule."

- symptoms: "Cannot expire objects — Object Lock prevents deletion"
  diagnosis: "Objects have retention periods that have not expired."
  resolution: "Wait for retention to expire. In Governance mode, use s3:BypassGovernanceRetention."

## Output Format

```yaml
root_cause: "expiration_issue — <specific_cause>"
evidence:
  - type: lifecycle_config
    content: "<expiration rules>"
  - type: versioning_status
    content: "<versioning state>"
severity: MEDIUM
mitigation:
  immediate: "Fix lifecycle expiration rules for versioned objects"
  long_term: "Implement NoncurrentVersionExpiration and delete marker cleanup"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying lifecycle expiration rules via put-bucket-lifecycle-configuration. Expiration rules permanently delete objects/versions, but rules can be removed before execution. Uses get-bucket-lifecycle-configuration and list-object-versions for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Expiration rule changes affect data retention or compliance requirements

## Rollback
- Pre-change: "Save current lifecycle configuration before modification"
- Verification: "Test that expiration rules target the intended objects after change"
- Revert: "Restore previous lifecycle configuration if change causes unintended deletions"

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
