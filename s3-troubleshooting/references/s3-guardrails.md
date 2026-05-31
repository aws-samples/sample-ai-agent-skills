# S3 Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any S3 issue.

## Guardrail 1: Access Evaluation is Multi-Layer
S3 access is evaluated across: IAM policy, bucket policy, ACL, S3 Block Public Access, VPC endpoint policy, SCP, and Object Lock. An explicit deny in ANY layer blocks access. Check ALL layers before concluding.

## Guardrail 2: Account-Level Block Public Access Overrides Bucket-Level
S3 Block Public Access has four independent settings at both account and bucket level. Account-level settings ALWAYS override bucket-level settings. A bucket cannot be made public if the account blocks it.

## Guardrail 3: Versioning Cannot Be Disabled
Once enabled, versioning can only be SUSPENDED, not disabled. Suspended versioning stops creating new versions but retains all existing versions. Existing versions must be explicitly deleted.

## Guardrail 4: Replication Does NOT Copy Existing Objects
Enabling replication only replicates NEW objects written after the rule is created. Existing objects require S3 Batch Replication. Delete markers are optionally replicated (not by default).

## Guardrail 5: Objects < 128 KB Don't Transition to IA
Lifecycle transitions to Standard-IA, One Zone-IA, or Intelligent-Tiering skip objects smaller than 128 KB. These objects remain in their current storage class.

## Guardrail 6: Bucket Policy Size Limit is 20 KB
Bucket policies cannot exceed 20 KB. Complex policies with many principals or conditions hit this limit. Use IAM policies or consolidate statements.

## Guardrail 7: Cross-Account Access Requires Both Sides
Cross-account S3 access requires BOTH: the source account IAM policy allowing the action AND the destination bucket policy granting access. Missing either side causes access denied.

## Guardrail 8: Delete Markers Are Not Objects
Delete markers are zero-byte version markers. They make the current version appear deleted but previous versions still exist. Deleting a delete marker restores the previous version.

## Guardrail 9: KMS Encryption Requires Key Permissions
SSE-KMS encrypted objects require kms:Decrypt on the KMS key for reads and kms:GenerateDataKey for writes. The S3 bucket policy alone is insufficient — KMS key policy must also allow access.

## Guardrail 10: Event Notification Uniqueness
Each event type (s3:ObjectCreated:*, etc.) can only have ONE destination per notification configuration. Configuring the same event to multiple destinations requires using different prefixes/suffixes as filters.

## Guardrail 11: Presigned URLs Inherit Creator Permissions
Presigned URLs use the permissions of the IAM principal that created them. If the principal's permissions are revoked or the credentials expire, the presigned URL stops working even if it hasn't expired.

## Guardrail 12: S3 Request Rate Limits Are Per-Prefix
S3 supports 5,500 GET/HEAD and 3,500 PUT/POST/DELETE per second per prefix. Spreading keys across multiple prefixes increases aggregate throughput. Sequential key naming no longer causes hot partitions.

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
