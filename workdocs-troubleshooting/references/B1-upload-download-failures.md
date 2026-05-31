---
title: "B1 — Upload/Download Failures"
description: "Diagnose WorkDocs upload and download failures"
status: active
severity: HIGH
triggers: ["upload failed", "download failed", "cannot upload", "file transfer error"]
owner: devops-agent
objective: "Resolve file upload and download issues including size limits, network errors, and permission problems"
context: "WorkDocs supports file uploads up to 5 GB via API. Web client and sync client may have lower practical limits."
---

## Phase 1 — Triage

MUST:
- Verify user status is Active: `aws workdocs describe-users --organization-id <org-id> --query 'Users[?Username==`<username>`]'`
- Check user storage quota and usage
- Verify the target folder exists and user has write permission: `aws workdocs describe-resource-permissions --resource-id <folder-id>`
- Check file size against limits (5 GB maximum)

SHOULD:
- Check WorkDocs activity feed for error events: `aws workdocs describe-activities --organization-id <org-id> --limit 20`
- Verify network connectivity to WorkDocs endpoint
- Check if the file type is blocked by admin policies

MAY:
- Review browser console logs for web client upload errors
- Check CloudTrail for InitiateDocumentVersionUpload failures

## Phase 2 — Remediate

MUST:
- Ensure user has sufficient storage quota for the upload
- Grant write permissions on the target folder if missing
- Reduce file size if exceeding the 5 GB limit

SHOULD:
- Use multipart upload for large files via the API
- Clear browser cache if web client uploads fail intermittently

MAY:
- Configure admin policies to allow blocked file types if appropriate
- Set up retry logic in API-based upload workflows

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user inspection"
  - "workdocs describe-resource-permissions: GREEN — read-only permission check"
  - "workdocs describe-activities: GREEN — read-only activity feed"
  - "workdocs add-resource-permissions: YELLOW — modifies sharing permissions"
```

## Escalation Conditions

- Uploads fail for all users suggesting a service-wide issue
- Large file uploads consistently fail at the same percentage
- Download failures affect specific documents that were previously accessible
- Upload failures persist after storage quota and permission fixes

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user identities
- `aws workdocs describe-resource-permissions`: MEDIUM — reveals access control lists
- `aws workdocs describe-activities`: MEDIUM — reveals user activity patterns
- Uploaded file content: HIGH — may contain sensitive business documents

## Prohibited Actions

- NEVER increase storage quotas beyond organizational policy limits
- NEVER grant permissions to folders without verifying the user should have access
- NEVER delete documents to free storage without user approval
- NEVER bypass file type restrictions without admin approval

## Phase 3 — Rollback

1. If permissions were granted during troubleshooting, remove them if not authorized: `aws workdocs remove-resource-permissions --resource-id <id> --principal-id <principal-id>`
2. If storage quota was increased, revert to original quota if the increase was temporary
3. If test files were uploaded, delete them to free storage
4. Restore any admin policy changes made during troubleshooting
5. Verify the user can perform normal upload/download operations after fixes

## Common Issues

- symptoms: "Upload fails with 'Storage limit exceeded'"
  diagnosis: "User has reached their storage quota."
  resolution: "Check quota: describe-users shows StorageRule. Increase quota or delete unnecessary files."

- symptoms: "Upload fails with 'Access denied' on folder"
  diagnosis: "User lacks write permission on the target folder."
  resolution: "Grant CONTRIBUTOR or CO_OWNER permission on the folder to the user."

## Output Format

```yaml
root_cause: "upload_download — <specific_cause>"
evidence:
  - type: transfer_error
    content: "<error details and user/folder info>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or quota"
  long_term: "Monitor storage usage and set alerts"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
