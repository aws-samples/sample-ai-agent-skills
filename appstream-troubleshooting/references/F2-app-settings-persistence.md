---
title: "F2 — App Settings Persistence Issues"
description: "Diagnose application settings persistence problems"
status: active
severity: MEDIUM
triggers:
  - "settings not persisting"
  - "app settings lost"
  - "preferences reset"
  - "settings persistence"
owner: devops-agent
objective: "Identify and resolve application settings persistence issues"
context: "App settings persistence saves Windows application settings to S3 and restores them in new sessions. It must be enabled on the stack. Only settings in supported Windows locations are captured. There is a size limit for persisted settings."
---

## Phase 1 — Triage

MUST:
- Check if app settings persistence is enabled on the stack: `aws appstream describe-stacks --names <stack-name> --query 'Stacks[*].ApplicationSettings'`
- Verify the settings group name is configured
- Check if the S3 bucket for app settings exists and is accessible
- Verify the user has had at least one session to save settings

SHOULD:
- Check if the application stores settings in supported Windows locations
- Verify the settings size is within limits
- Check S3 bucket for the user's settings files

MAY:
- Test with a simple application that stores settings in a known location
- Check CloudTrail for S3 access errors

## Phase 2 — Remediate

MUST:
- Enable app settings persistence on the stack:
  `aws appstream update-stack --name <stack-name> --application-settings Enabled=true,SettingsGroup=<group-name>`
- Ensure the S3 bucket is accessible
- Verify the IAM role has S3 permissions

SHOULD:
- Configure the settings group name for organizing settings
- Test settings persistence with a known application

MAY:
- Monitor S3 storage usage for app settings
- Document which applications support settings persistence

## Common Issues

- symptoms: "Settings reset every session"
  diagnosis: "App settings persistence not enabled on the stack."
  resolution: "Enable app settings persistence: update-stack with ApplicationSettings."

- symptoms: "Some app settings persist but others don't"
  diagnosis: "Application stores settings in unsupported Windows locations."
  resolution: "Only settings in supported locations (AppData, etc.) are captured."

- symptoms: "Settings persistence enabled but not working"
  diagnosis: "S3 bucket access issue or settings size exceeded."
  resolution: "Check S3 bucket permissions and settings size."

## Output Format

```yaml
root_cause: "app_settings_persistence_issue — <specific_cause>"
evidence:
  - type: stack_settings
    content: "<app settings persistence configuration>"
  - type: s3_access
    content: "<S3 bucket accessibility>"
  - type: settings_data
    content: "<user settings in S3>"
severity: MEDIUM
mitigation:
  immediate: "Enable and fix app settings persistence"
  long_term: "Document supported settings locations"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable app settings persistence | YELLOW | Stack configuration change |
| Ensure S3 bucket is accessible | GREEN | Verification |
| Verify IAM role has S3 permissions | GREEN | Permission check |
| Configure settings group name | GREEN | Configuration |
| Test settings persistence | GREEN | Non-disruptive testing |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **App settings**: User application preferences and configurations — CONFIDENTIAL
- **S3 bucket**: Settings storage location and access — INTERNAL
- **Settings group**: Organizational grouping for settings — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER disable app settings persistence without notifying users
- NEVER delete settings data from S3 without backup

## Phase 3 — Rollback

1. Disable app settings persistence if it was newly enabled and caused issues
2. Revert settings group name to previous value
3. Restore S3 bucket permissions to previous state
4. Verify existing user settings are not corrupted
5. Communicate settings changes to affected users

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
