---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Health Dashboard issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "health dashboard problem"
  - "health issue"
  - "something wrong with health dashboard"
  - "health not working"
owner: devops-agent
objective: "Systematically diagnose unspecified Health Dashboard issues"
context: "This catch-all runbook provides a systematic approach when the specific issue category is unclear. It covers initial data collection, common checks, and routing to specific runbooks."
---

## Phase 1 — Triage

MUST:
- Check for active events: `aws health describe-events --filter '{"eventStatusCodes":["open","upcoming"]}' --region us-east-1`
- Verify Health API access (Business/Enterprise Support required)
- Check EventBridge rules for health notifications: `aws events list-rules --region us-east-1`
- Verify IAM permissions for health:* actions

SHOULD:
- Check organizational health if using Organizations
- Review recent event history
- Verify support plan level

MAY:
- Check AWS status page for service-wide issues
- Review CloudTrail for Health API activity

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If event interpretation issue → A1-A3
- If notification issue → B1-B2
- If organization issue → C1-C2
- If API issue → D1-D2
- If response issue → E1-E2

SHOULD:
- Document findings and route to specific runbook
- Collect all relevant evidence before escalating

MAY:
- Contact AWS Support for Health-specific issues
- Review AWS documentation for recent Health service changes

## Common Issues

- symptoms: "Health Dashboard not working (unspecified)"
  diagnosis: "Multiple possible causes. Systematic triage required."
  resolution: "Follow Phase 1 triage to identify category, then route to specific runbook."

- symptoms: "Cannot access Health Dashboard features"
  diagnosis: "Support plan limitation or IAM permission issue."
  resolution: "Check support plan (Business/Enterprise for API). Verify IAM permissions."

- symptoms: "Health events seem incomplete"
  diagnosis: "Events may be still populating or older than 90-day retention."
  resolution: "Re-check events later. Events older than 90 days are not available."

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: health_status
    content: "<active events and API access status>"
  - type: triage_findings
    content: "<initial diagnostic results>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook based on triage"
  long_term: "Implement comprehensive health monitoring and notification"
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
- Never dismiss or archive health events without documenting their resolution
- Never disable EventBridge rules for health notifications in production accounts
- Never ignore scheduled maintenance events without planning for impact
- Never delete organizational health view configurations without stakeholder approval
- Never modify health event filters that may suppress critical notifications

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable EventBridge notification rules if health alerts stop flowing
- Restore previous event filter configurations if critical events are missed
- Revert organizational view delegation changes if visibility is lost

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
