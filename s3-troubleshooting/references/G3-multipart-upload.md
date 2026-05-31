---
title: "G3 — Multipart Upload Failures"
description: "Diagnose multipart upload failures including part size, completion, and ETag issues"
status: active
severity: MEDIUM
triggers:
  - "Multipart upload failed"
  - "Part size"
  - "CompleteMultipartUpload error"
  - "ETag mismatch"
  - "EntityTooSmall"
owner: devops-agent
objective: "Identify and fix multipart upload failures"
context: "Multipart upload allows uploading objects up to 5 TB in parts. Part size must be 5 MB to 5 GB (except the last part). Maximum 10,000 parts per upload. CompleteMultipartUpload requires the correct part numbers and ETags. Incomplete uploads leave orphaned parts that incur storage costs."
---

## Phase 1 — Triage

MUST:
- List active multipart uploads: `aws s3api list-multipart-uploads --bucket <bucket>`
- List parts of a specific upload: `aws s3api list-parts --bucket <bucket> --key <key> --upload-id <upload-id>`
- Check the error message (EntityTooSmall, InvalidPart, NoSuchUpload)
- Verify part sizes are within limits (5 MB minimum except last part)

SHOULD:
- Check if the upload ID is still valid (uploads can expire or be aborted)
- Verify ETags match between uploaded parts and completion request
- Check for network timeouts during part uploads

MAY:
- Check application logs for retry behavior on failed parts
- Verify S3 permissions include s3:PutObject, s3:AbortMultipartUpload, s3:ListMultipartUploadParts

## Phase 2 — Remediate

MUST:
- Fix part sizes to be at least 5 MB (except the last part)
- Ensure CompleteMultipartUpload includes correct part numbers and ETags
- Abort failed uploads to avoid orphaned parts: `aws s3api abort-multipart-upload --bucket <bucket> --key <key> --upload-id <upload-id>`

SHOULD:
- Implement retry logic for individual part uploads
- Use the AWS SDK's managed upload which handles multipart automatically
- Add lifecycle rule for AbortIncompleteMultipartUpload (see E3)

MAY:
- Increase part size for large files to reduce the number of parts
- Use parallel part uploads for better throughput

## Common Issues

- symptoms: "EntityTooSmall error on CompleteMultipartUpload"
  diagnosis: "One or more parts (except the last) are smaller than 5 MB."
  resolution: "Ensure all parts except the last are at least 5 MB."

- symptoms: "InvalidPart error on completion"
  diagnosis: "ETag in the completion request does not match the uploaded part."
  resolution: "Use the ETag returned by each UploadPart response in the completion request."

- symptoms: "NoSuchUpload error"
  diagnosis: "The upload was aborted or expired (lifecycle rule or manual abort)."
  resolution: "Start a new multipart upload. Check for lifecycle AbortIncompleteMultipartUpload rules."

## Output Format

```yaml
root_cause: "multipart_upload — <specific_cause>"
evidence:
  - type: upload_parts
    content: "<part listing with sizes and ETags>"
  - type: error_message
    content: "<specific error>"
severity: MEDIUM
mitigation:
  immediate: "Fix part sizes or ETags and retry the upload"
  long_term: "Use SDK managed uploads and add lifecycle cleanup rules"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses list-multipart-uploads and list-parts. Remediation involves aborting failed uploads and fixing application upload logic. Abort-multipart-upload removes orphaned data only. No bucket security changes. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Upload failures affect production data pipelines

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
