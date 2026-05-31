---
title: "A2 — Contact Execution Failures"
description: "Diagnose satellite contact execution problems"
status: active
severity: CRITICAL
triggers: ["contact failed", "FAILED status", "no data received"]
owner: devops-agent
objective: "Identify and resolve contact execution failures"
context: "Execution fails when: dataflow endpoint not ready, antenna config wrong, weather interference, or satellite issue."
---

## Phase 1 — Triage

MUST:
- Check contact status: `aws groundstation describe-contact --contact-id <id>`
- Verify dataflow endpoint EC2 instances were running during contact
- Check mission profile configuration
- Review contact metrics for signal quality

SHOULD:
- Check weather conditions at ground station location
- Verify antenna configuration matches satellite
- Check CloudWatch for contact-related metrics

## Phase 2 — Remediate

MUST:
- Ensure EC2 dataflow endpoints are running before contact
- Fix antenna configuration to match satellite
- Schedule replacement contact

SHOULD:
- Start EC2 instances well before contact window
- Monitor contact progress in real-time

## Output Format

```yaml
root_cause: "contact_execution — <specific_cause>"
evidence:
  - type: contact_status
    content: "<contact details>"
severity: CRITICAL
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
