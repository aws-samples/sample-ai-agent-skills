---
title: "G2 — Access Control"
description: "Diagnose WorkSpace access control and authorization issues"
status: active
severity: HIGH
triggers:
  - "access denied workspace"
  - "IP access control"
  - "workspace authorization"
  - "device access restriction"
owner: devops-agent
objective: "Resolve WorkSpace access control configuration issues"
context: "WorkSpaces access is controlled by IP access control groups (allowlists), device type restrictions, and directory-level settings. IP access control groups are allowlists — an empty group blocks all connections."
---

## Phase 1 — Triage

MUST:
- Check IP access control groups: `aws workspaces describe-ip-groups`
- Check directory access control settings: `aws workspaces describe-workspace-directories --query 'Directories[].WorkspaceAccessProperties'`
- Verify IP group associations with directory: `aws workspaces describe-workspace-directories --query 'Directories[].{DirId:DirectoryId,IpGroups:ipGroupIds}'`
- Check if user's client IP is in the allowed ranges

SHOULD:
- Verify device type access settings (Windows, macOS, iOS, Android, Linux, Web)
- Check if certificate-based authentication is required
- Review CloudTrail for access denied events

MAY:
- Check if trusted device policies are configured via Group Policy
- Verify client certificate deployment status

## Phase 2 — Remediate

MUST:
- Add client IP CIDR to IP access control group if legitimate access is blocked
- Remove IP access control group association if no IP restrictions are needed
- Enable required device types in workspace access properties

SHOULD:
- Document IP access control group rules and review periodically
- Configure device type access based on security policy
- Set up monitoring for access denied events

MAY:
- Implement certificate-based authentication for additional security
- Use conditional access policies for granular control

## Common Issues

- symptoms: "All users suddenly cannot connect"
  diagnosis: "Empty IP access control group associated with directory blocks all connections."
  resolution: "Add IP rules to the group or disassociate the empty group from the directory."

- symptoms: "User blocked from specific device type"
  diagnosis: "Device type access disabled in workspace access properties."
  resolution: "Enable the device type: `aws workspaces modify-workspace-access-properties --resource-id <dir-id> --workspace-access-properties DeviceTypeWindows=ALLOW`"

- symptoms: "Remote users cannot connect after office IP change"
  diagnosis: "IP access control group has outdated CIDR ranges."
  resolution: "Update IP access control group with new CIDR: `aws workspaces authorize-ip-rules --group-id <group-id> --user-rules ipRule=<cidr>,ruleDesc=<desc>`"

## Output Format

```yaml
root_cause: "access_control — <specific_cause>"
evidence:
  - type: ip_groups
    content: "<IP access control group rules>"
  - type: access_properties
    content: "<device type and access settings>"
severity: HIGH
mitigation:
  immediate: "Fix IP access control rules or device type settings"
  long_term: "Implement access control review process and monitoring"
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
