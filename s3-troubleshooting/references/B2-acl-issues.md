---
title: "B2 — ACL Issues"
description: "Diagnose issues with S3 ACLs including BucketOwnerEnforced and legacy ACL problems"
status: active
severity: MEDIUM
triggers:
  - "ACL"
  - "AccessControlList"
  - "BucketOwnerEnforced"
  - "canned ACL"
  - "object ownership"
owner: devops-agent
objective: "Identify ACL-related access issues and migrate to modern access controls"
context: "New S3 buckets default to BucketOwnerEnforced, which disables ACLs entirely. Legacy buckets may still use ACLs. Three ownership settings exist: BucketOwnerEnforced (ACLs disabled), BucketOwnerPreferred, and ObjectWriter. AWS recommends disabling ACLs and using bucket policies instead."
---

## Phase 1 — Triage

MUST:
- Check object ownership setting: `aws s3api get-bucket-ownership-controls --bucket <bucket>`
- Check bucket ACL: `aws s3api get-bucket-acl --bucket <bucket>`
- If ACLs are enabled, check object ACL: `aws s3api get-object-acl --bucket <bucket> --key <key>`
- Check if PutObject requests include ACL headers (x-amz-acl, x-amz-grant-*)

SHOULD:
- Identify if the bucket uses legacy ACLs for access control
- Check if cross-account uploads are setting object ownership correctly
- Verify Block Public Access settings for public ACLs

MAY:
- List objects with non-default ACLs using S3 Inventory
- Check if applications depend on canned ACLs (public-read, bucket-owner-full-control)

## Phase 2 — Remediate

MUST:
- For new buckets: keep BucketOwnerEnforced (default) and use bucket policies
- For legacy buckets: migrate ACL-based access to bucket policies before enabling BucketOwnerEnforced
- Fix applications that send ACL headers when BucketOwnerEnforced is set (they get 400 errors)

SHOULD:
- Enable BucketOwnerEnforced on all buckets after migrating to policies
- Remove ACL headers from application PutObject calls
- Update cross-account upload workflows to not require ACLs

MAY:
- Use S3 Batch Operations to reset object ACLs before migration
- Audit all buckets for ACL usage before organization-wide migration

## Common Issues

- symptoms: "PutObject fails with AccessControlListNotSupported"
  diagnosis: "BucketOwnerEnforced is enabled but the request includes ACL headers."
  resolution: "Remove x-amz-acl or x-amz-grant-* headers from the request."

- symptoms: "Cross-account uploader cannot read their own objects"
  diagnosis: "ObjectWriter ownership means the uploader owns the object, but bucket policy only grants bucket owner access."
  resolution: "Switch to BucketOwnerEnforced so the bucket owner always owns objects."

- symptoms: "Public read access stopped working"
  diagnosis: "IgnorePublicAcls was enabled, making public-read ACLs ineffective."
  resolution: "Use a bucket policy for public access instead of ACLs, or disable IgnorePublicAcls."

## Output Format

```yaml
root_cause: "acl_issue — <specific_cause>"
evidence:
  - type: ownership_controls
    content: "<ownership setting>"
  - type: bucket_acl
    content: "<ACL grants>"
severity: MEDIUM
mitigation:
  immediate: "Fix the ACL or ownership configuration"
  long_term: "Migrate to BucketOwnerEnforced and bucket policies"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying ACLs and ownership controls via put-bucket-acl and put-bucket-ownership-controls. ACL changes can alter who has read/write access to bucket contents. |

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
