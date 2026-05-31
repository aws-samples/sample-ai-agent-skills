---
title: "G1 — Home Folder Issues"
description: "Diagnose AppStream 2.0 home folder problems"
status: active
severity: MEDIUM
triggers:
  - "home folder not working"
  - "cannot save files"
  - "home folder access"
  - "persistent storage"
owner: devops-agent
objective: "Identify and resolve home folder issues"
context: "Home folders provide S3-backed persistent storage for users. They are enabled on the stack. Each user gets a folder in the S3 bucket. Files saved to the home folder persist across sessions. The S3 bucket is created automatically."
---

## Phase 1 — Triage

MUST:
- Check if home folders are enabled on the stack: `aws appstream describe-stacks --names <stack-name> --query 'Stacks[*].StorageConnectors'`
- Verify the S3 bucket exists: `aws s3 ls | grep appstream2`
- Check if the user can access their home folder in the session
- Verify IAM permissions for S3 access

SHOULD:
- Check S3 bucket policy for access restrictions
- Verify the user's folder exists in the bucket: `aws s3 ls s3://<bucket>/<user-hash>/`
- Check S3 bucket encryption settings

MAY:
- Check S3 storage usage per user
- Verify S3 bucket lifecycle policies

## Phase 2 — Remediate

MUST:
- Enable home folders on the stack if not enabled
- Ensure the S3 bucket is accessible and not blocked by bucket policy
- Fix IAM permissions for S3 access

SHOULD:
- Verify the S3 bucket is in the same region as the AppStream stack
- Check for S3 bucket policy restrictions

MAY:
- Set up S3 lifecycle policies for cost management
- Monitor S3 storage usage

## Common Issues

- symptoms: "Home folder not visible in session"
  diagnosis: "Home folders not enabled on the stack."
  resolution: "Enable home folders via stack storage connectors."

- symptoms: "Cannot save files to home folder"
  diagnosis: "S3 bucket access denied or bucket policy blocking writes."
  resolution: "Check S3 bucket policy and IAM permissions."

- symptoms: "Files saved but not available in next session"
  diagnosis: "Files saved to local storage instead of home folder."
  resolution: "Ensure users save to the 'My Files > Home Folder' location."

## Output Format

```yaml
root_cause: "home_folder_issue — <specific_cause>"
evidence:
  - type: stack_storage
    content: "<storage connector configuration>"
  - type: s3_bucket
    content: "<S3 bucket status and permissions>"
  - type: user_folder
    content: "<user folder in S3>"
severity: MEDIUM
mitigation:
  immediate: "Fix home folder configuration"
  long_term: "Monitor S3 storage and set up lifecycle policies"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable home folders on stack | GREEN | Feature enablement |
| Ensure S3 bucket is accessible | GREEN | Verification |
| Fix IAM permissions for S3 | YELLOW | IAM policy change |
| Verify S3 bucket in same region | GREEN | Configuration check |
| Set up S3 lifecycle policies | GREEN | Cost management |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Home folder data**: User files and documents — CONFIDENTIAL
- **S3 bucket**: Persistent storage location — INTERNAL
- **User folder paths**: User-specific storage paths — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER delete S3 home folder buckets without user data backup
- NEVER modify S3 bucket policies without understanding impact on user access

## Phase 3 — Rollback

1. Revert S3 bucket policy changes
2. Revert IAM permission changes
3. Verify users can access their home folders
4. Restore any accidentally deleted user data from S3 versioning
5. Confirm home folder functionality in a test session

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
