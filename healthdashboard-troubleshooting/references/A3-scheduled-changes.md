---
title: "A3 — Scheduled Changes"
description: "Manage and plan for scheduled maintenance events"
status: active
severity: MEDIUM
triggers:
  - "scheduled maintenance"
  - "upcoming change"
  - "planned maintenance"
  - "maintenance notification"
owner: devops-agent
objective: "Plan and prepare for scheduled maintenance events"
context: "Scheduled change events notify about planned maintenance (RDS patching, EC2 retirement, EBS maintenance). Some allow rescheduling. Events include start time but may not have fixed end time. Affected entities identify specific resources."
---

## Phase 1 — Triage

MUST:
- List upcoming scheduled changes: `aws health describe-events --filter '{"eventTypeCategories":["scheduledChange"],"eventStatusCodes":["upcoming"]}' --region us-east-1`
- Get event details: `aws health describe-event-details --event-arns <event-arn> --region us-east-1`
- Identify affected resources: `aws health describe-affected-entities --filter '{"eventArns":["<event-arn>"]}' --region us-east-1`
- Check maintenance window timing and duration

SHOULD:
- Determine if maintenance can be rescheduled
- Assess impact on production workloads
- Check if affected resources have redundancy

MAY:
- Review historical maintenance patterns for the service
- Check if maintenance aligns with your change management windows

## Phase 2 — Remediate

MUST:
- Plan for maintenance window (communicate to stakeholders)
- Ensure redundancy for affected resources during maintenance
- Take pre-maintenance snapshots/backups where applicable

SHOULD:
- Reschedule maintenance if possible and timing conflicts with business needs
- For EC2 instance retirement: migrate to new instance before deadline
- For RDS maintenance: apply during preferred maintenance window

MAY:
- Automate pre-maintenance preparation (snapshots, failover testing)
- Set up post-maintenance validation checks

## Common Issues

- symptoms: "EC2 instance scheduled for retirement"
  diagnosis: "Underlying hardware issue requires instance migration."
  resolution: "Stop and start instance to migrate to new hardware. Or launch replacement."

- symptoms: "RDS maintenance window conflict"
  diagnosis: "Scheduled maintenance overlaps with business hours."
  resolution: "Modify maintenance window: `aws rds modify-db-instance --db-instance-identifier <id> --preferred-maintenance-window <window>`"

- symptoms: "Maintenance completed but resource still impaired"
  diagnosis: "Post-maintenance issue or resource needs restart."
  resolution: "Check resource health. Restart if needed. Contact AWS Support if issue persists."

## Output Format

```yaml
root_cause: "scheduled_changes — <specific_cause>"
evidence:
  - type: event_details
    content: "<scheduled change details and timeline>"
  - type: affected_resources
    content: "<resources affected by maintenance>"
severity: MEDIUM
mitigation:
  immediate: "Plan for maintenance window and ensure redundancy"
  long_term: "Automate maintenance preparation and post-maintenance validation"
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
