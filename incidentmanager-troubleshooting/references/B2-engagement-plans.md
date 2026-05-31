---
title: "B2 — Engagement Plan Issues"
description: "Diagnose Incident Manager engagement plan problems including contact routing and timing"
status: active
severity: MEDIUM
triggers:
  - "engagement plan"
  - "engagement not starting"
  - "wrong contacts engaged"
  - "engagement timing"
owner: devops-agent
objective: "Resolve engagement plan issues and ensure correct contact notification"
context: "Engagement plans define which contacts are notified and in what order. They can be simple (direct contact) or complex (escalation plans with stages). Common issues include wrong contact references, timing misconfigurations, and plan not being triggered."
---

## Phase 1 — Triage

MUST:
- Check engagement plan in response plan: `aws ssm-incidents get-response-plan --arn <plan-arn>`
- List active engagements: `aws ssm-contacts list-engagements`
- Verify contacts referenced in the plan exist and are active
- Check if the incident impact level matches the engagement configuration

SHOULD:
- Verify engagement plan stages and timing
- Check if contacts have activated channels
- Review engagement history for past incidents

MAY:
- Test engagement manually: `aws ssm-contacts start-engagement --contact-id <contact-arn> --subject "Test" --content "Test"`
- Check CloudTrail for engagement events

## Phase 2 — Remediate

MUST:
- Fix contact references in the response plan
- Ensure all referenced contacts have activated channels
- Verify impact level configuration matches engagement plan

SHOULD:
- Test engagement plans with simulated incidents
- Document engagement procedures

## Output Format

```yaml
root_cause: "engagement_plan — <specific_cause>"
evidence:
  - type: response_plan_engagements
    content: "<engagement configuration>"
  - type: active_engagements
    content: "<current engagements>"
severity: MEDIUM
mitigation:
  immediate: "Fix engagement plan configuration"
  long_term: "Test engagement plans regularly"
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
