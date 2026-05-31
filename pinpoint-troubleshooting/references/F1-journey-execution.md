---
title: "F1 — Journey Execution Issues"
description: "Diagnose journey execution failures and unexpected behavior"
status: active
severity: HIGH
triggers:
  - "journey not starting"
  - "journey execution failed"
  - "participants not entering journey"
  - "journey stuck"
owner: devops-agent
objective: "Identify and resolve journey execution issues"
context: "Journeys orchestrate multi-step customer engagement. They require ACTIVE status, valid entry conditions, and properly configured activities. Maximum duration is 18 months."
---

## Phase 1 — Triage

MUST:
- Check journey status: `aws pinpoint get-journey --application-id <app-id> --journey-id <journey-id>`
- List journey runs: `aws pinpoint get-journey-execution-metrics --application-id <app-id> --journey-id <journey-id>`
- Verify journey state is ACTIVE
- Check entry condition segment or event trigger

SHOULD:
- Verify journey start/end dates
- Check if journey has reached participant limit
- Review activity configuration for errors

MAY:
- Check CloudTrail for journey state changes
- Review per-activity execution metrics

## Phase 2 — Remediate

MUST:
- Set journey to ACTIVE state if paused/draft
- Fix entry condition (segment must exist and have endpoints)
- Ensure journey dates are valid (start in past/present, end in future)

SHOULD:
- Test journey with small segment before full launch
- Monitor journey execution metrics after launch

MAY:
- Implement journey lifecycle automation

## Common Issues

- symptoms: "Journey is ACTIVE but no participants entering"
  diagnosis: "Entry segment is empty or event trigger not firing."
  resolution: "Verify entry segment has endpoints. Check event trigger configuration."

- symptoms: "Journey stopped accepting new participants"
  diagnosis: "Journey reached end date or 18-month maximum duration."
  resolution: "Create new journey. Cannot extend past maximum duration."

## Output Format

```yaml
root_cause: "journey_execution — <specific_cause>"
evidence:
  - type: journey_status
    content: "<journey configuration and state>"
  - type: execution_metrics
    content: "<journey execution metrics>"
severity: HIGH
mitigation:
  immediate: "Fix journey state or entry conditions"
  long_term: "Implement journey monitoring and lifecycle management"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — journeys process customer data across multiple touchpoints
- Service Disruption Risk: HIGH — journey failures affect customer experience
- Compliance Impact: HIGH — journeys must comply with all messaging regulations

## Escalation Conditions
- Journey execution completely stops
- Journey sends messages to wrong customers
- Journey activities execute out of order
- Journey causes excessive message volume
- Journey fails to respect opt-out preferences

## Data Sensitivity
- Customer journey data — RESTRICTED
- Journey configurations — INTERNAL
- Activity execution logs — INTERNAL
- Customer touchpoint data — RESTRICTED
- Never expose customer journey paths or PII

## Prohibited Actions
- Do NOT modify active journeys without pausing first
- Do NOT create journeys without proper opt-out handling
- Do NOT bypass journey validation
- Do NOT ignore journey execution errors
- Do NOT create journeys that violate messaging frequency limits
- Do NOT expose journey configurations externally

## Phase 3 — Rollback
- Pause the journey immediately if issues detected
- Restore previous journey configuration
- Revert activity changes
- Verify journey execution after changes
- Process opt-out requests from affected customers
- Document journey issues and compliance impact

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
