---
title: "E2 — Storage Limits"
description: "Diagnose WorkDocs storage limit and quota issues"
status: active
severity: MEDIUM
triggers: ["storage full", "quota exceeded", "cannot upload storage", "storage limit reached"]
owner: devops-agent
objective: "Resolve storage quota issues including per-user limits, organizational capacity, and storage optimization"
context: "WorkDocs storage is per-user based on user type. Quotas can be adjusted by admins within plan limits."
---

## Phase 1 — Triage

MUST:
- Check user storage usage: `aws workdocs describe-users --organization-id <org-id> --query 'Users[?Username==`<username>`].{Storage:StorageRule,Type:Type}'`
- List all users with storage usage to identify heavy users: `aws workdocs describe-users --organization-id <org-id> --query 'Users[*].{Username:Username,Type:Type,Storage:StorageRule}'`
- Verify user type and associated default quota

SHOULD:
- Check Recycle Bin for recoverable space (deleted files still count against quota)
- Identify large files or folders consuming storage
- Check organizational total storage allocation

MAY:
- Review upload activity to identify rapid storage growth: `aws workdocs describe-activities --organization-id <org-id> --limit 50`
- Check if versioned documents are consuming excessive storage

## Phase 2 — Remediate

MUST:
- Increase user storage quota if within organizational limits
- Empty Recycle Bin to reclaim space from deleted files
- Identify and remove duplicate or unnecessary files

SHOULD:
- Upgrade user type for users who need more storage (USER → POWERUSER)
- Set up storage usage alerts before quotas are reached

MAY:
- Implement document retention policies to manage storage growth
- Archive old documents to S3 for long-term storage

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user and storage inspection"
  - "workdocs update-user (quota): YELLOW — changes storage allocation"
  - "Empty Recycle Bin: YELLOW — permanently deletes recoverable content"
  - "workdocs delete-document: RED — permanently removes documents"
```

## Escalation Conditions

- Organizational storage limit reached and cannot be increased
- Storage usage does not decrease after deleting files (sync delay)
- Multiple users hitting quotas simultaneously suggesting organizational growth
- Storage costs exceed budget and optimization is needed

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user storage usage patterns
- `aws workdocs describe-activities`: MEDIUM — reveals upload patterns
- File and folder listings: MEDIUM — reveals document names and structure

## Prohibited Actions

- NEVER permanently delete user files to free storage without explicit user approval
- NEVER reduce storage quotas below current usage — this blocks all uploads
- NEVER empty another user's Recycle Bin without their knowledge
- NEVER disable versioning organization-wide to save storage

## Phase 3 — Rollback

1. If storage quota was reduced, restore to previous level: `aws workdocs update-user --user-id <id> --storage-rule <original-rule>`
2. If files were deleted to free space, check Recycle Bin for recovery (within retention period)
3. If user type was changed for storage reasons, revert if not authorized
4. Verify user can upload after quota changes
5. Document storage changes and new quota allocations

## Common Issues

- symptoms: "User cannot upload — 'Storage limit exceeded'"
  diagnosis: "User has reached their per-user storage quota."
  resolution: "Increase quota, empty Recycle Bin, or delete unnecessary files. Check with: describe-users."

- symptoms: "Storage usage high despite deleting many files"
  diagnosis: "Deleted files are in Recycle Bin and still count against quota."
  resolution: "Empty the Recycle Bin to permanently delete files and reclaim storage."

## Output Format

```yaml
root_cause: "storage_limits — <specific_cause>"
evidence:
  - type: storage_issue
    content: "<user quota, usage, and organizational capacity>"
severity: MEDIUM
mitigation:
  immediate: "Free storage or increase quota"
  long_term: "Implement storage monitoring and retention policies"
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
