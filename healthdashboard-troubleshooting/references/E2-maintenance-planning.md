---
title: "E2 — Maintenance Planning"
description: "Plan and coordinate scheduled maintenance from Health events"
status: active
severity: MEDIUM
triggers:
  - "maintenance planning"
  - "scheduled maintenance"
  - "maintenance window"
  - "upcoming maintenance"
owner: devops-agent
objective: "Effectively plan for and manage scheduled maintenance events"
context: "Scheduled change events provide advance notice of planned maintenance. Some maintenance can be rescheduled. Planning involves identifying affected resources, assessing impact, preparing redundancy, and communicating to stakeholders."
---

## Phase 1 — Triage

MUST:
- List upcoming scheduled changes: `aws health describe-events --filter '{"eventTypeCategories":["scheduledChange"],"eventStatusCodes":["upcoming"]}' --region us-east-1`
- Get maintenance details: `aws health describe-event-details --event-arns <event-arn> --region us-east-1`
- Identify affected resources: `aws health describe-affected-entities --filter '{"eventArns":["<event-arn>"]}' --region us-east-1`
- Check maintenance window timing and assess business impact

SHOULD:
- Determine if maintenance is reschedulable
- Check if affected resources have redundancy (Multi-AZ, replicas)
- Review maintenance prerequisites (backups, snapshots)

MAY:
- Check historical maintenance patterns for the service
- Coordinate with change management process

## Phase 2 — Remediate

MUST:
- Create maintenance plan with timeline and responsibilities
- Ensure backups/snapshots are current before maintenance
- Communicate maintenance schedule to affected teams

SHOULD:
- Reschedule maintenance to preferred window if possible
- Test failover before maintenance window
- Prepare rollback plan

MAY:
- Automate pre-maintenance preparation (snapshots, health checks)
- Set up post-maintenance validation automation

## Common Issues

- symptoms: "Maintenance window too short to prepare"
  diagnosis: "Some maintenance events have short notice periods."
  resolution: "Set up EventBridge notifications for early awareness. Maintain always-ready redundancy."

- symptoms: "Cannot reschedule maintenance"
  diagnosis: "Not all maintenance types support rescheduling."
  resolution: "Check event details for rescheduling options. Prepare for the scheduled window."

- symptoms: "Post-maintenance issues"
  diagnosis: "Resource may need restart or configuration update after maintenance."
  resolution: "Run post-maintenance validation. Restart services if needed. Check resource health."

## Output Format

```yaml
root_cause: "maintenance_planning — <specific_cause>"
evidence:
  - type: scheduled_events
    content: "<upcoming maintenance events and timeline>"
  - type: affected_resources
    content: "<resources requiring maintenance preparation>"
severity: MEDIUM
mitigation:
  immediate: "Create maintenance plan and prepare resources"
  long_term: "Implement automated maintenance preparation and validation"
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
