---
title: "E1 — Ground Station Location Selection"
description: "Diagnose ground station location and coverage issues"
status: active
severity: MEDIUM
triggers: ["no coverage", "wrong location", "limited passes"]
owner: devops-agent
objective: "Identify and resolve ground station location issues"
---

## Phase 1 — Triage

MUST:
- List available ground stations: `aws groundstation list-ground-stations`
- Check satellite orbit type (LEO, MEO, GEO)
- Verify ground station location covers satellite orbit
- Check available contact windows per location

SHOULD:
- Compare contact windows across multiple locations
- Check antenna capabilities at each location
- Verify region availability for data delivery

## Phase 2 — Remediate

MUST:
- Choose ground stations that cover satellite orbit
- Use multiple locations for more contact windows
- Verify antenna type matches requirements

SHOULD:
- Plan ground station strategy based on orbit analysis
- Document coverage analysis

## Output Format

```yaml
root_cause: "location_selection — <specific_cause>"
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
