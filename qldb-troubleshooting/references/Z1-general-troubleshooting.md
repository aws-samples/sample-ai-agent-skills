---
title: "Z1 — General QLDB Troubleshooting"
description: "Catch-all runbook for unclassified QLDB issues"
status: active
severity: MEDIUM
triggers: ["QLDB not working", "something wrong with QLDB"]
owner: devops-agent
objective: "Systematically triage unclassified QLDB issues"
---

## Phase 1 — Triage

MUST:
- List ledgers: `aws qldb list-ledgers`
- Check ledger status: `aws qldb describe-ledger --name <name>`
- Check streams: `aws qldb list-journal-kinesis-streams-for-ledger --ledger-name <name>`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=qldb.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions
- Check CloudWatch metrics
- Verify service health

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B2, C1-C2, D1-D2, E1-E2, F1-F2, or G1-G2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — troubleshooting may involve inspecting ledger configurations
- Service Disruption Risk: LOW — diagnostic operations are read-only
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- QLDB service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows QLDB service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Ledger configurations — INTERNAL
- CloudWatch metrics — INTERNAL
- Error messages — INTERNAL
- Query patterns — INTERNAL
- Never expose ledger data during troubleshooting

## Prohibited Actions
- Do NOT make changes to production ledgers without approval
- Do NOT run destructive commands during initial diagnosis
- Do NOT share ledger details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside QLDB service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore ledger configurations to last known good state
- Document all troubleshooting steps and outcomes
- Verify ledger health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
