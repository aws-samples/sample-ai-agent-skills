---
title: "B1 — Client Connection"
description: "Diagnose WorkSpaces client connection failures"
status: active
severity: HIGH
triggers:
  - "cannot connect to workspace"
  - "client connection failed"
  - "workspace unreachable"
  - "connection timed out"
owner: devops-agent
objective: "Identify and resolve WorkSpaces client connectivity issues"
context: "Client connection failures stem from network restrictions, IP access control groups, security group rules, WorkSpace state issues, or client version incompatibility. PCoIP uses port 4172, WSP uses port 4195."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace state: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].{State:State,Protocol:WorkspaceProperties.Protocols}'`
- Check connection status: `aws workspaces describe-connection-status --workspace-id <id>`
- Verify IP access control groups: `aws workspaces describe-ip-groups`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[].IpPermissions'`

SHOULD:
- Verify WorkSpace is not in ADMIN_MAINTENANCE or SUSPENDED state
- Check if auto-stop WorkSpace needs to be started: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].{RunningMode:WorkspaceProperties.RunningMode,State:State}'`
- Test DNS resolution of WorkSpaces gateway endpoints

MAY:
- Check client-side firewall logs for blocked connections
- Verify NACLs allow traffic on required ports

## Phase 2 — Remediate

MUST:
- Ensure WorkSpace is in AVAILABLE state
- Open required ports: 4172 (PCoIP) or 4195 (WSP) TCP+UDP, and 443 TCP
- Fix IP access control group rules if blocking client IP

SHOULD:
- Update WorkSpaces client to latest version
- Start auto-stop WorkSpace if in STOPPED state: `aws workspaces start-workspaces --start-workspace-requests WorkspaceId=<id>`

MAY:
- Switch protocol from PCoIP to WSP for better connectivity through firewalls

## Common Issues

- symptoms: "Client shows 'Unable to connect' error"
  diagnosis: "IP access control group blocking client IP, or required ports not open."
  resolution: "Check IP access control groups. Verify ports 4172/4195 and 443 are open."

- symptoms: "Connection works intermittently"
  diagnosis: "Auto-stop WorkSpace stopping after idle timeout, or network instability."
  resolution: "Check running mode. Consider switching to AlwaysOn if consistent access needed."

- symptoms: "Client connects but screen is black"
  diagnosis: "WorkSpace OS issue or graphics driver problem."
  resolution: "Reboot WorkSpace via API: `aws workspaces reboot-workspaces --reboot-workspace-requests WorkspaceId=<id>`"

## Output Format

```yaml
root_cause: "client_connection — <specific_cause>"
evidence:
  - type: workspace_state
    content: "<workspace state and protocol>"
  - type: connection_status
    content: "<last connection status and timestamp>"
severity: HIGH
mitigation:
  immediate: "Fix network rules, IP access groups, or workspace state"
  long_term: "Monitor connection metrics and set up health alerts"
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
