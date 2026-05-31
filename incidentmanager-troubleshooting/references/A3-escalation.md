---
title: "A3 — Escalation Issues"
description: "Diagnose Incident Manager escalation plan problems including timing and contact routing"
status: active
severity: HIGH
triggers:
  - "escalation plan"
  - "escalation not working"
  - "wrong contacts notified"
  - "escalation timing"
  - "contacts not engaged"
owner: devops-agent
objective: "Resolve escalation plan issues and ensure correct contact engagement"
context: "Escalation plans define staged contact engagement based on time. Each stage has a duration and set of contacts. Acknowledgment stops escalation. Common issues include wrong timing, unactivated channels, and acknowledgment confusion."
---

## Phase 1 — Triage

MUST:
- Get escalation plan details: `aws ssm-contacts get-contact --contact-id <escalation-plan-arn>`
- Check contact channel activation status for each contact in the plan
- Verify escalation plan stages and timing
- Check active engagements: `aws ssm-contacts list-engagements`

SHOULD:
- Verify the response plan references the correct escalation plan
- Check if incident was acknowledged (stops escalation)
- Verify impact level matches escalation plan configuration

MAY:
- Check engagement history: `aws ssm-contacts list-page-receipts --page-id <page-id>`
- Test escalation with a simulated incident

## Phase 2 — Remediate

MUST:
- If channels unactivated, have recipients activate them
- If timing wrong, update escalation plan stage durations
- If wrong contacts, update escalation plan stages

SHOULD:
- Test escalation plans regularly
- Document escalation procedures for the team

## Output Format

```yaml
root_cause: "escalation — <specific_cause>"
evidence:
  - type: escalation_plan
    content: "<escalation plan details>"
  - type: engagement_status
    content: "<active engagements>"
severity: HIGH
mitigation:
  immediate: "Fix escalation plan configuration or channel activation"
  long_term: "Test escalation plans monthly and maintain channel activation"
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
- Never delete response plans that are actively referenced by CloudWatch alarms
- Never modify escalation plans during active incidents
- Never remove contact channels without confirming alternative notification paths exist
- Never delete incident records or post-incident analysis data
- Never disable automation runbooks tied to active response plans without backup procedures

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous response plan configuration if incident detection breaks
- Revert escalation plan changes if notifications fail to reach responders
- Re-enable previous contact channel settings if engagement plans stop working

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
