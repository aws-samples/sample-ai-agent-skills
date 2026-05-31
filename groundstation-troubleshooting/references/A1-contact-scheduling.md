---
title: "A1 — Contact Scheduling Failures"
description: "Diagnose satellite contact scheduling and reservation failures"
status: active
severity: HIGH
triggers: ["cannot schedule", "no available contacts", "scheduling error"]
owner: devops-agent
objective: "Identify and resolve contact scheduling failures"
context: "Scheduling fails when: no pass windows available, mission profile invalid, satellite not onboarded, or ground station unavailable."
---

## Phase 1 — Triage

MUST:
- List available contacts: `aws groundstation list-contacts --status AVAILABLE --satellite-id <id> --start-time <start> --end-time <end> --ground-station <gs-id>`
- Verify satellite is onboarded: `aws groundstation list-satellites`
- Check mission profile: `aws groundstation get-mission-profile --mission-profile-id <id>`
- Verify ground station availability

SHOULD:
- Expand time range to find available windows
- Check multiple ground station locations
- Verify satellite orbital parameters

MAY:
- Check for conflicting reservations
- Verify account limits for concurrent contacts

## Phase 2 — Remediate

MUST:
- Onboard satellite if not configured
- Fix mission profile configuration
- Choose available pass windows

SHOULD:
- Schedule contacts well in advance
- Use multiple ground stations for more windows

## Output Format

```yaml
root_cause: "contact_scheduling — <specific_cause>"
evidence:
  - type: available_contacts
    content: "<contact windows>"
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
