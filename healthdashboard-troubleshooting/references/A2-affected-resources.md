---
title: "A2 — Affected Resources"
description: "Identify resources affected by health events"
status: active
severity: HIGH
triggers:
  - "which resources affected"
  - "affected entities"
  - "impacted resources"
  - "resource impact"
owner: devops-agent
objective: "Identify specific resources affected by health events"
context: "Affected entities are available for account-specific events only. Public events do not list specific resources. Entity lists may be incomplete initially and are populated over time as AWS identifies impacted resources."
---

## Phase 1 — Triage

MUST:
- Check if event is public or account-specific: `aws health describe-event-details --event-arns <event-arn> --region us-east-1`
- Get affected entities: `aws health describe-affected-entities --filter '{"eventArns":["<event-arn>"]}' --region us-east-1`
- Check entity status values (IMPAIRED, UNIMPAIRED, UNKNOWN, PENDING)
- Verify entity list completeness (may be populated over time)

SHOULD:
- Cross-reference affected entities with your resource inventory
- Check if affected resources are in production or non-production
- Monitor entity list for updates as event progresses

MAY:
- Check CloudWatch metrics for affected resources
- Verify resource health independently of the health event

## Phase 2 — Remediate

MUST:
- For public events: assume all resources in affected region/service may be impacted
- For account-specific events: review listed affected entities
- Verify resource health independently using service-specific tools
- Plan mitigation based on affected resource criticality

SHOULD:
- Implement failover for critical affected resources
- Communicate affected resource list to application owners
- Monitor affected resources for recovery

MAY:
- Automate affected resource identification and notification
- Create runbooks for common affected resource scenarios

## Common Issues

- symptoms: "No affected entities listed"
  diagnosis: "Public event (no specific entities) or entity list not yet populated."
  resolution: "For public events, check all resources in affected region. For account events, re-check later."

- symptoms: "Entity status shows UNKNOWN"
  diagnosis: "AWS has not yet determined the impact on this specific resource."
  resolution: "Check resource health independently. Re-check entity status later."

- symptoms: "Affected entity list keeps growing"
  diagnosis: "AWS is progressively identifying impacted resources."
  resolution: "Monitor entity list for updates. Plan response for newly identified resources."

## Output Format

```yaml
root_cause: "affected_resources — <specific_cause>"
evidence:
  - type: affected_entities
    content: "<list of affected resources and status>"
  - type: event_type
    content: "<public vs account-specific event>"
severity: HIGH
mitigation:
  immediate: "Identify and protect critical affected resources"
  long_term: "Implement automated affected resource tracking and response"
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
