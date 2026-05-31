---
title: "G2 — App Settings Storage Issues"
description: "Diagnose AppStream 2.0 application settings storage problems"
status: active
severity: LOW
triggers:
  - "app settings storage"
  - "settings S3 bucket"
  - "settings backup"
  - "settings size"
owner: devops-agent
objective: "Identify and resolve app settings storage issues"
context: "App settings are stored in S3 buckets managed by AppStream. Settings are organized by settings group and user. There are size limits for persisted settings. S3 bucket policies and encryption affect settings storage."
---

## Phase 1 — Triage

MUST:
- Check app settings configuration: `aws appstream describe-stacks --names <stack-name> --query 'Stacks[*].ApplicationSettings'`
- Verify the S3 bucket for app settings exists and is accessible
- Check the settings group name
- Verify S3 bucket permissions

SHOULD:
- Check S3 storage usage for app settings
- Verify S3 bucket encryption settings
- Check for S3 bucket lifecycle policies

MAY:
- List user settings in S3: `aws s3 ls s3://<bucket>/<settings-group>/`
- Check S3 access logs for errors

## Phase 2 — Remediate

MUST:
- Ensure the S3 bucket is accessible and not blocked
- Fix S3 bucket policy if it restricts AppStream access
- Verify the settings group name is consistent

SHOULD:
- Set up S3 lifecycle policies to manage storage costs
- Monitor S3 storage usage

MAY:
- Back up app settings periodically
- Document settings storage architecture

## Common Issues

- symptoms: "App settings S3 bucket not found"
  diagnosis: "Bucket was deleted or never created."
  resolution: "Re-enable app settings persistence. AppStream creates the bucket automatically."

- symptoms: "Settings storage growing unexpectedly"
  diagnosis: "Large application settings or many users."
  resolution: "Set up S3 lifecycle policies. Monitor per-user storage."

- symptoms: "Settings from wrong group applied"
  diagnosis: "Settings group name mismatch between stacks."
  resolution: "Use consistent settings group names across stacks that should share settings."

## Output Format

```yaml
root_cause: "app_settings_storage_issue — <specific_cause>"
evidence:
  - type: settings_config
    content: "<app settings configuration>"
  - type: s3_bucket
    content: "<S3 bucket status>"
  - type: storage_usage
    content: "<S3 storage usage>"
severity: LOW
mitigation:
  immediate: "Fix S3 bucket access"
  long_term: "Set up lifecycle policies and monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure S3 bucket is accessible | GREEN | Verification |
| Fix S3 bucket policy | YELLOW | S3 policy change |
| Verify settings group name | GREEN | Configuration check |
| Set up S3 lifecycle policies | GREEN | Cost management |
| Back up app settings | GREEN | Data protection |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **App settings data**: User application preferences — CONFIDENTIAL
- **S3 bucket**: Settings storage location — INTERNAL
- **Settings group**: Settings organization structure — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER delete app settings S3 buckets without backup
- NEVER change settings group names without understanding impact on existing settings

## Phase 3 — Rollback

1. Revert S3 bucket policy changes
2. Restore previous settings group name if changed
3. Verify app settings are loading correctly in sessions
4. Restore settings from backup if data was lost
5. Confirm settings persistence in a test session

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
