---
title: "C1 — Latency"
description: "Diagnose high latency and poor responsiveness in WorkSpaces sessions"
status: active
severity: MEDIUM
triggers:
  - "workspace slow"
  - "high latency"
  - "input lag"
  - "workspace unresponsive"
owner: devops-agent
objective: "Identify and resolve WorkSpaces session latency issues"
context: "WorkSpaces latency is affected by network round-trip time, protocol choice, compute resources, and client-side factors. Optimal experience requires <100ms RTT. WSP generally provides lower latency than PCoIP."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace compute type and resource utilization: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].WorkspaceProperties.ComputeTypeName'`
- Check CloudWatch latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/WorkSpaces --metric-name SessionLatency --dimensions Name=WorkspaceId,Value=<id> --start-time <iso> --end-time <iso> --period 300 --statistics Average`
- Verify WorkSpace health: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].State'`
- Check connection status for last known latency: `aws workspaces describe-connection-status --workspace-id <id>`

SHOULD:
- Check CPU and memory utilization via CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/WorkSpaces --metric-name InSessionLatency --dimensions Name=WorkspaceId,Value=<id> --start-time <iso> --end-time <iso> --period 300 --statistics p99`
- Verify protocol (WSP vs PCoIP) and UDP availability
- Check if the user is connecting from a distant geographic region

MAY:
- Run network path analysis between client and WorkSpaces gateway
- Check for bandwidth-intensive applications running on the WorkSpace

## Phase 2 — Remediate

MUST:
- Upgrade compute type if CPU/memory constrained: `aws workspaces modify-workspace-properties --workspace-id <id> --workspace-properties ComputeTypeName=PERFORMANCE`
- Ensure UDP is not blocked for optimal streaming performance

SHOULD:
- Migrate to WSP protocol for lower latency
- Recommend users connect from regions closer to the WorkSpaces deployment
- Reboot WorkSpace to clear resource exhaustion: `aws workspaces reboot-workspaces --reboot-workspace-requests WorkspaceId=<id>`

MAY:
- Deploy WorkSpaces in additional regions closer to user populations
- Configure QoS policies on corporate network for WorkSpaces traffic

## Common Issues

- symptoms: "Consistent 200ms+ latency"
  diagnosis: "User connecting from a distant region or network path has high RTT."
  resolution: "Deploy WorkSpaces in a region closer to users. Check network path for bottlenecks."

- symptoms: "Latency spikes during business hours"
  diagnosis: "Compute resources exhausted or network congestion."
  resolution: "Upgrade compute type. Check for resource-intensive applications."

- symptoms: "Mouse and keyboard input lag"
  diagnosis: "UDP blocked, forcing TCP-only streaming with higher latency."
  resolution: "Open UDP ports (4172 for PCoIP, 4195 for WSP) in firewall."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<session latency statistics>"
  - type: compute_config
    content: "<compute type and resource utilization>"
severity: MEDIUM
mitigation:
  immediate: "Upgrade compute type, fix UDP connectivity, or reboot"
  long_term: "Deploy in regions closer to users, standardize on WSP"
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
