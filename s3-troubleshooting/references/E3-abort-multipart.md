---
title: "E3 — Incomplete Multipart Uploads"
description: "Diagnose orphaned multipart upload parts and configure lifecycle cleanup"
status: active
severity: MEDIUM
triggers:
  - "Incomplete multipart upload"
  - "Orphaned parts"
  - "Multipart cleanup"
  - "AbortIncompleteMultipartUpload"
owner: devops-agent
objective: "Identify and clean up incomplete multipart uploads that waste storage"
context: "Multipart uploads that are not completed or aborted leave orphaned parts that incur storage costs. These parts are invisible to list-objects but visible to list-multipart-uploads. A lifecycle rule with AbortIncompleteMultipartUpload automatically cleans them up."
---

## Phase 1 — Triage

MUST:
- List incomplete multipart uploads: `aws s3api list-multipart-uploads --bucket <bucket>`
- Check for lifecycle cleanup rule: `aws s3api get-bucket-lifecycle-configuration --bucket <bucket>`
- Look for AbortIncompleteMultipartUpload in lifecycle rules
- Check storage costs — orphaned parts are billed as Standard storage

SHOULD:
- List parts of a specific upload: `aws s3api list-parts --bucket <bucket> --key <key> --upload-id <id>`
- Check how old the incomplete uploads are
- Identify the source application that is not completing uploads

MAY:
- Use S3 Storage Lens to identify buckets with high incomplete multipart upload costs
- Check CloudTrail for CreateMultipartUpload events without CompleteMultipartUpload

## Phase 2 — Remediate

MUST:
- Add lifecycle rule to abort incomplete multipart uploads:
  ```json
  {
    "Rules": [{
      "ID": "AbortIncompleteMultipartUploads",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
  }
  ```
- Manually abort old uploads if needed: `aws s3api abort-multipart-upload --bucket <bucket> --key <key> --upload-id <id>`

SHOULD:
- Fix the application to properly complete or abort multipart uploads
- Set DaysAfterInitiation based on the longest expected upload duration
- Apply the cleanup rule to all buckets as a best practice

MAY:
- Use a script to abort all current incomplete uploads before the lifecycle rule takes effect
- Monitor with CloudWatch S3 metrics for ongoing incomplete uploads

## Common Issues

- symptoms: "Storage costs higher than expected but bucket appears small"
  diagnosis: "Orphaned multipart upload parts are consuming storage."
  resolution: "List multipart uploads, abort them, and add lifecycle cleanup rule."

- symptoms: "list-objects shows nothing but storage is billed"
  diagnosis: "Incomplete multipart parts are not visible in list-objects."
  resolution: "Use list-multipart-uploads to find and abort orphaned uploads."

- symptoms: "Lifecycle cleanup rule exists but old parts remain"
  diagnosis: "DaysAfterInitiation is set too high, or the rule was recently added."
  resolution: "Lifecycle runs asynchronously. Wait for the next evaluation cycle."

## Output Format

```yaml
root_cause: "incomplete_multipart — <specific_cause>"
evidence:
  - type: multipart_uploads
    content: "<list of incomplete uploads>"
  - type: lifecycle_config
    content: "<cleanup rule status>"
severity: MEDIUM
mitigation:
  immediate: "Abort orphaned uploads and add lifecycle cleanup rule"
  long_term: "Apply AbortIncompleteMultipartUpload to all buckets"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying lifecycle rules and aborting multipart uploads via put-bucket-lifecycle-configuration and abort-multipart-upload. Aborting uploads removes incomplete parts permanently, but these are orphaned data. Uses list-multipart-uploads for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Lifecycle changes affect active upload workflows

## Rollback
- Pre-change: "Save current lifecycle configuration before modification"
- Verification: "Test that abort rules do not affect active multipart uploads"
- Revert: "Restore previous lifecycle configuration if change causes unintended upload aborts"

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
