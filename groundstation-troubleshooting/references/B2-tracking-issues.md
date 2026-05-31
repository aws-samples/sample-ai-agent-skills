---
title: "B2 — Tracking Issues"
description: "Diagnose antenna tracking and pointing problems"
status: active
severity: HIGH
triggers: ["tracking lost", "signal lost", "pointing error"]
owner: devops-agent
objective: "Identify and resolve antenna tracking issues"
context: "Tracking issues occur when: autotrack signal too weak, TLE data stale, tracking config wrong, or satellite maneuver."
---

## Phase 1 — Triage

MUST:
- Check tracking configuration in mission profile
- Verify autotrack vs programmatic tracking mode
- Check contact status: `aws groundstation describe-contact --contact-id <id>`
- Review contact metrics for signal quality

SHOULD:
- Verify satellite TLE (Two-Line Element) data is current
- Check if satellite performed recent maneuver
- Verify tracking frequency matches satellite beacon

## Phase 2 — Remediate

MUST:
- Update TLE data if stale
- Switch tracking mode if autotrack signal insufficient
- Verify tracking frequency configuration

SHOULD:
- Use programmatic tracking as fallback
- Monitor signal quality during contacts

## Output Format

```yaml
root_cause: "tracking — <specific_cause>"
severity: HIGH
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
