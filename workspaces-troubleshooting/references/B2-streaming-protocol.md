---
title: "B2 — Streaming Protocol"
description: "Diagnose PCoIP and WSP streaming protocol issues"
status: active
severity: MEDIUM
triggers:
  - "PCoIP error"
  - "WSP connection failed"
  - "streaming quality poor"
  - "protocol mismatch"
owner: devops-agent
objective: "Resolve WorkSpaces streaming protocol configuration and performance issues"
context: "WorkSpaces supports PCoIP and WSP protocols. WSP offers better performance, Web Access support, and smart card authentication. Protocol migration requires a WorkSpace reboot. Each protocol uses different ports and has different client requirements."
---

## Phase 1 — Triage

MUST:
- Check current protocol: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].WorkspaceProperties.Protocols'`
- Verify protocol-specific ports are open (PCoIP: 4172, WSP: 4195): `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check client version compatibility with the configured protocol
- Verify WorkSpace bundle supports the desired protocol: `aws workspaces describe-workspace-bundles --bundle-ids <bundle-id>`

SHOULD:
- Check CloudWatch streaming metrics: `aws cloudwatch get-metric-statistics --namespace AWS/WorkSpaces --metric-name SessionLaunchTime --dimensions Name=WorkspaceId,Value=<id> --start-time <iso> --end-time <iso> --period 300 --statistics Average`
- Verify UDP connectivity for optimal streaming performance
- Test with both TCP and UDP to isolate protocol issues

MAY:
- Compare performance between PCoIP and WSP on the same WorkSpace
- Check if corporate firewall is blocking UDP traffic

## Phase 2 — Remediate

MUST:
- Open correct ports for the configured protocol
- Ensure client version supports the WorkSpace protocol
- Reboot WorkSpace after protocol migration

SHOULD:
- Migrate to WSP for better performance and Web Access support
- Enable UDP for optimal streaming quality
- Update client to latest version

MAY:
- Configure protocol-specific settings (compression, frame rate) via Group Policy

## Common Issues

- symptoms: "Web Access not working"
  diagnosis: "WorkSpace is configured with PCoIP. Web Access requires WSP."
  resolution: "Migrate to WSP: `aws workspaces modify-workspace-properties --workspace-id <id> --workspace-properties Protocols=WSP`"

- symptoms: "Poor video quality over WSP"
  diagnosis: "UDP blocked by firewall, forcing TCP fallback with higher latency."
  resolution: "Open UDP port 4195 in firewall. WSP performs best with UDP."

- symptoms: "Protocol migration failed"
  diagnosis: "WorkSpace not in AVAILABLE state or bundle doesn't support target protocol."
  resolution: "Ensure WorkSpace is AVAILABLE. Verify bundle supports WSP."

## Output Format

```yaml
root_cause: "streaming_protocol — <specific_cause>"
evidence:
  - type: protocol_config
    content: "<current protocol and port status>"
  - type: streaming_metrics
    content: "<session launch time and quality metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix port configuration or protocol settings"
  long_term: "Standardize on WSP protocol for all WorkSpaces"
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
