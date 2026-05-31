---
title: "Z1 — General Ground Station Troubleshooting"
description: "Catch-all runbook for unclassified Ground Station issues"
status: active
severity: MEDIUM
triggers: ["Ground Station not working", "something wrong with Ground Station"]
owner: devops-agent
objective: "Systematically triage unclassified Ground Station issues"
---

## Phase 1 — Triage

MUST:
- List ground stations: `aws groundstation list-ground-stations`
- List satellites: `aws groundstation list-satellites`
- List configs: `aws groundstation list-configs`
- List mission profiles: `aws groundstation list-mission-profiles`

SHOULD:
- Check recent contacts: `aws groundstation list-contacts --status COMPLETED --start-time <start> --end-time <end>`
- Check CloudTrail for errors
- Verify IAM permissions

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B2, C1-C2, D1-D2, or E1-E2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
severity: MEDIUM
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never cancel scheduled satellite contacts without confirming mission impact
- Never modify mission profiles during active contacts
- Never delete dataflow endpoint groups with upcoming scheduled contacts
- Never change antenna configurations without validating against satellite requirements
- Never alter ground station location settings for active mission profiles

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reschedule cancelled contacts at the earliest available window
- Restore previous mission profile configuration if contact execution fails
- Revert dataflow endpoint changes if data delivery is disrupted

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
