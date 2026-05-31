---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for WorkSpaces issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "workspace problem"
  - "workspace issue"
  - "something wrong with workspace"
  - "workspace not working"
owner: devops-agent
objective: "Systematically diagnose unspecified WorkSpaces issues"
context: "This catch-all runbook provides a systematic approach when the specific issue category is unclear. It covers initial data collection, common checks, and routing to specific runbooks."
---

## Phase 1 — Triage

MUST:
- Get WorkSpace details: `aws workspaces describe-workspaces --workspace-id <id>`
- Check WorkSpace state and health
- Check directory status: `aws workspaces describe-workspace-directories`
- Check connection status: `aws workspaces describe-connection-status --workspace-id <id>`

SHOULD:
- Check CloudWatch metrics for anomalies: `aws cloudwatch get-metric-statistics --namespace AWS/WorkSpaces --metric-name Available --dimensions Name=WorkspaceId,Value=<id> --start-time <iso> --end-time <iso> --period 300 --statistics Average`
- Review recent CloudTrail events for the WorkSpace
- Check if the issue correlates with a maintenance window

MAY:
- Check AWS Health Dashboard for service events
- Review VPC and networking configuration
- Check IAM permissions for the calling identity

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If provisioning issue → A1-A3
- If connectivity issue → B1-B3
- If performance issue → C1-C2
- If directory issue → D1-D2
- If networking issue → E1-E2
- If management issue → F1-F2
- If security issue → G1-G2

SHOULD:
- Document findings and route to specific runbook
- Collect all relevant evidence before escalating

MAY:
- Reboot WorkSpace as a general recovery step: `aws workspaces reboot-workspaces --reboot-workspace-requests WorkspaceId=<id>`
- Check AWS Support cases for known issues

## Common Issues

- symptoms: "WorkSpace not working (unspecified)"
  diagnosis: "Multiple possible causes. Systematic triage required."
  resolution: "Follow Phase 1 triage to identify category, then route to specific runbook."

- symptoms: "Multiple WorkSpaces affected simultaneously"
  diagnosis: "Likely infrastructure issue — directory, VPC, or AWS service event."
  resolution: "Check directory health, VPC configuration, and AWS Health Dashboard."

- symptoms: "Single WorkSpace intermittently failing"
  diagnosis: "Could be auto-stop behavior, maintenance window, or resource exhaustion."
  resolution: "Check running mode, maintenance schedule, and compute utilization."

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: workspace_state
    content: "<workspace details and state>"
  - type: triage_findings
    content: "<initial diagnostic results>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook based on triage"
  long_term: "Implement comprehensive monitoring and alerting"
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
