---
title: "F2 — Maintenance"
description: "Diagnose WorkSpace maintenance window and update issues"
status: active
severity: MEDIUM
triggers:
  - "maintenance window"
  - "ADMIN_MAINTENANCE"
  - "workspace updates"
  - "workspace patching"
owner: devops-agent
objective: "Resolve WorkSpace maintenance and update-related issues"
context: "AlwaysOn WorkSpaces have a maintenance window (default Sunday 00:00-04:00 UTC) for Windows updates. During maintenance, WorkSpaces enter ADMIN_MAINTENANCE state and are inaccessible. AutoStop WorkSpaces are maintained when started."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace state: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].{State:State,RunningMode:WorkspaceProperties.RunningMode}'`
- Check maintenance window configuration: `aws workspaces describe-workspace-directories --query 'Directories[].WorkspaceCreationProperties'`
- Verify if current time falls within maintenance window
- Check if WorkSpace is in ADMIN_MAINTENANCE state

SHOULD:
- Review maintenance window settings for the directory
- Check if Windows Update is causing extended maintenance
- Verify WorkSpace returns to AVAILABLE after maintenance window

MAY:
- Check CloudWatch for maintenance-related state changes
- Review Windows Update history on the WorkSpace

## Phase 2 — Remediate

MUST:
- Wait for maintenance window to complete if WorkSpace is in ADMIN_MAINTENANCE
- Do not attempt to force-start a WorkSpace during maintenance

SHOULD:
- Adjust maintenance window to off-peak hours for the user's timezone
- Configure Windows Update policies via Group Policy to control update behavior
- Communicate maintenance schedules to users

MAY:
- Disable automatic maintenance for specific WorkSpaces if business-critical
- Stagger maintenance windows across different directories

## Common Issues

- symptoms: "WorkSpace unavailable during business hours"
  diagnosis: "Maintenance window overlaps with user's working hours due to timezone difference."
  resolution: "Adjust maintenance window to align with off-peak hours in user's timezone."

- symptoms: "WorkSpace stuck in ADMIN_MAINTENANCE for hours"
  diagnosis: "Large Windows Update or failed update causing extended maintenance."
  resolution: "Wait up to 4 hours. If still stuck, reboot via API: `aws workspaces reboot-workspaces --reboot-workspace-requests WorkspaceId=<id>`"

- symptoms: "AutoStop WorkSpace takes long to start"
  diagnosis: "Pending updates applied during startup, extending boot time."
  resolution: "Expected behavior. Updates are applied when AutoStop WorkSpaces start. Consider AlwaysOn mode."

## Output Format

```yaml
root_cause: "maintenance — <specific_cause>"
evidence:
  - type: workspace_state
    content: "<workspace state and running mode>"
  - type: maintenance_config
    content: "<maintenance window settings>"
severity: MEDIUM
mitigation:
  immediate: "Wait for maintenance completion or adjust window"
  long_term: "Align maintenance windows with off-peak hours and communicate schedules"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- WorkSpaces contain user desktop data and files
- Directory integration reveals Active Directory structure
- Connection logs contain user access patterns
- Classification: CONFIDENTIAL - user workspace data and directory information

## Prohibited Actions

- NEVER terminate or rebuild WorkSpaces without user data backup confirmation
- NEVER modify directory service configurations during business hours
- NEVER change MFA settings without user notification and transition period
- NEVER modify VPC or subnet configurations for running WorkSpaces without maintenance window

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
