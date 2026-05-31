---
title: "A1 — Event Interpretation"
description: "Diagnose and interpret AWS Health Dashboard events"
status: active
severity: HIGH
triggers:
  - "what does this event mean"
  - "health event interpretation"
  - "event impact assessment"
  - "service event meaning"
owner: devops-agent
objective: "Correctly interpret health events and assess their impact"
context: "Health events have three categories: issue (active disruption), accountNotification (account-specific info), and scheduledChange (planned maintenance). Each has different urgency. Event details include description, affected services, regions, and timeline."
---

## Phase 1 — Triage

MUST:
- Get event details: `aws health describe-event-details --event-arns <event-arn> --region us-east-1`
- Identify event category (issue, accountNotification, scheduledChange)
- Check event status (upcoming, open, closed)
- Determine affected service and region

SHOULD:
- Check affected entities: `aws health describe-affected-entities --filter '{"eventArns":["<event-arn>"]}' --region us-east-1`
- Review event timeline (start time, end time, last updated)
- Check if event is public (service-wide) or account-specific

MAY:
- Check for related events in the same timeframe
- Review event type code for additional context
- Check AWS status page for broader context

## Phase 2 — Remediate

MUST:
- Assess impact based on event category and affected resources
- For issues: check if your resources are in the affected region/service
- For scheduled changes: plan for the maintenance window
- For account notifications: review and take recommended actions

SHOULD:
- Communicate event impact to stakeholders
- Document affected resources and expected timeline
- Set up monitoring for event status changes

MAY:
- Implement automated response based on event type
- Create incident tickets for issue-category events

## Common Issues

- symptoms: "Confused about event severity"
  diagnosis: "Event category determines urgency: issue > scheduledChange > accountNotification."
  resolution: "Check event category. Issues require immediate attention. Scheduled changes need planning."

- symptoms: "Event says 'operational issue' but services seem fine"
  diagnosis: "Event may affect specific AZs or resource types, not all resources."
  resolution: "Check affected entities and region. Your resources may not be impacted."

- symptoms: "Multiple events for same service"
  diagnosis: "Separate events for different aspects of the same incident."
  resolution: "Review all related events. Each may have different affected resources."

## Output Format

```yaml
root_cause: "event_interpretation — <specific_cause>"
evidence:
  - type: event_details
    content: "<event category, status, and description>"
  - type: affected_scope
    content: "<affected service, region, and entities>"
severity: HIGH
mitigation:
  immediate: "Assess impact and communicate to stakeholders"
  long_term: "Set up automated event processing and response"
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
