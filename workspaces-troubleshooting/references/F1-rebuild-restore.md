---
title: "F1 — Rebuild and Restore"
description: "Diagnose WorkSpace rebuild and restore operation failures"
status: active
severity: HIGH
triggers:
  - "rebuild failed"
  - "restore workspace"
  - "workspace recovery"
  - "snapshot restore"
owner: devops-agent
objective: "Resolve WorkSpace rebuild and restore operation issues"
context: "Rebuild restores root volume from bundle image and user volume from last snapshot. Restore reverts user volume to the last snapshot. Snapshots are taken every 12 hours. Data after the last snapshot is lost during rebuild/restore."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace state: `aws workspaces describe-workspaces --workspace-id <id>`
- Check snapshot availability: `aws workspaces describe-workspace-snapshots --workspace-id <id>`
- Verify the WorkSpace is not in a transitional state (PENDING, REBOOTING, REBUILDING)
- Check if rebuild/restore was recently initiated: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RebuildWorkspaces --max-results 5`

SHOULD:
- Verify bundle still exists for rebuild: `aws workspaces describe-workspace-bundles --bundle-ids <bundle-id>`
- Check if the WorkSpace has been running long enough for a snapshot (first snapshot after 12 hours)
- Review CloudTrail for error details on failed operations

MAY:
- Check if there are IAM permission issues preventing the operation
- Verify subnet has available IPs for the rebuilt WorkSpace

## Phase 2 — Remediate

MUST:
- Ensure WorkSpace is in AVAILABLE, STOPPED, or ERROR state before rebuild
- Warn user about data loss (data after last snapshot will be lost)
- Initiate rebuild: `aws workspaces rebuild-workspaces --rebuild-workspace-requests WorkspaceId=<id>`

SHOULD:
- Take a manual snapshot before rebuild if WorkSpace is accessible
- Document the last snapshot timestamp for user communication
- Monitor rebuild progress via describe-workspaces

MAY:
- Restore user volume only (instead of full rebuild) to preserve root volume customizations: `aws workspaces restore-workspace --workspace-id <id>`

## Common Issues

- symptoms: "Rebuild stuck in REBUILDING state"
  diagnosis: "Rebuild typically takes 30-60 minutes. May take longer for large volumes."
  resolution: "Wait up to 2 hours. If still stuck, contact AWS Support."

- symptoms: "User data missing after rebuild"
  diagnosis: "Data created after the last automatic snapshot was lost during rebuild."
  resolution: "Explain snapshot timing (every 12 hours). Recommend regular backups to S3 or OneDrive."

- symptoms: "Cannot rebuild — bundle not found"
  diagnosis: "Original bundle was deleted or is no longer available."
  resolution: "Create a new bundle or use a different compatible bundle for rebuild."

## Output Format

```yaml
root_cause: "rebuild_restore — <specific_cause>"
evidence:
  - type: workspace_state
    content: "<workspace state and snapshot availability>"
  - type: operation_history
    content: "<CloudTrail events for rebuild/restore>"
severity: HIGH
mitigation:
  immediate: "Fix workspace state or bundle availability, then retry"
  long_term: "Implement regular backup strategy and user data protection"
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
