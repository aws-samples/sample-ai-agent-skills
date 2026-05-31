---
title: "C2 — Bandwidth"
description: "Diagnose bandwidth constraints and streaming quality issues"
status: active
severity: MEDIUM
triggers:
  - "poor video quality"
  - "blurry workspace"
  - "bandwidth throttling"
  - "streaming degraded"
owner: devops-agent
objective: "Resolve bandwidth-related WorkSpaces performance issues"
context: "WorkSpaces streaming requires 300 Kbps minimum, 1-2 Mbps recommended for standard use, and 4+ Mbps for graphics-intensive work. Bandwidth is affected by screen resolution, content type, and network capacity."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace protocol and compute type: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].{Protocol:WorkspaceProperties.Protocols,Compute:WorkspaceProperties.ComputeTypeName}'`
- Monitor bandwidth metrics: `aws cloudwatch get-metric-statistics --namespace AWS/WorkSpaces --metric-name SessionDisconnect --dimensions Name=WorkspaceId,Value=<id> --start-time <iso> --end-time <iso> --period 300 --statistics Sum`
- Check if multiple WorkSpaces share a constrained network path
- Verify internet gateway or NAT gateway throughput

SHOULD:
- Check VPC flow logs for dropped packets: `aws ec2 describe-flow-logs --filter Name=resource-id,Values=<eni-id>`
- Verify no bandwidth-limiting policies on the network path
- Check WorkSpace display settings (resolution, color depth)

MAY:
- Run bandwidth test from client to WorkSpaces region
- Check if content redirection is enabled to offload media playback

## Phase 2 — Remediate

MUST:
- Ensure minimum 1 Mbps bandwidth per WorkSpace session
- Reduce display resolution if bandwidth is constrained

SHOULD:
- Enable WSP protocol for better bandwidth efficiency
- Configure bandwidth throttling limits via Group Policy
- Enable content redirection for video playback

MAY:
- Upgrade network infrastructure between client and AWS
- Deploy WorkSpaces closer to users to reduce bandwidth overhead

## Common Issues

- symptoms: "Screen appears blurry or pixelated"
  diagnosis: "Insufficient bandwidth causing aggressive compression."
  resolution: "Ensure 1+ Mbps available bandwidth. Reduce resolution if needed."

- symptoms: "Frequent session disconnects"
  diagnosis: "Bandwidth drops below minimum threshold causing session drops."
  resolution: "Check network path for congestion. Ensure stable 300+ Kbps connection."

- symptoms: "Video playback stuttering on WorkSpace"
  diagnosis: "Streaming video within a remote desktop doubles bandwidth requirement."
  resolution: "Enable browser content redirection to play video locally on client device."

## Output Format

```yaml
root_cause: "bandwidth — <specific_cause>"
evidence:
  - type: bandwidth_metrics
    content: "<session bandwidth and disconnect statistics>"
  - type: network_config
    content: "<VPC and network path configuration>"
severity: MEDIUM
mitigation:
  immediate: "Reduce resolution, enable content redirection, or fix network path"
  long_term: "Upgrade network capacity and deploy closer to users"
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
