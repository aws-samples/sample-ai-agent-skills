---
title: "F2 — Maintenance Events"
description: "Diagnose and prepare for MediaConnect maintenance events and service disruptions"
status: active
severity: MEDIUM
triggers: ["maintenance window", "scheduled maintenance", "flow interruption", "AWS Health event", "service disruption"]
owner: devops-agent
objective: "Minimize impact of maintenance events on live video delivery"
context: "AWS schedules maintenance for MediaConnect infrastructure. Flows may experience brief interruptions during maintenance. Source failover with dual-AZ sources maintains availability."
---

## Phase 1 — Triage

MUST:
- Check AWS Health Dashboard for scheduled events: `aws health describe-events --filter eventTypeCategories=scheduledChange,services=MEDIACONNECT`
- Check flow status: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query '{Status:Flow.Status,AvailabilityZone:Flow.AvailabilityZone}'`
- Verify source failover is configured: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.SourceFailoverConfig'`

SHOULD:
- Check if maintenance affects the flow's AZ
- Review event details for expected duration and impact
- Verify both failover sources are healthy

MAY:
- Check historical maintenance patterns for the region
- Review AWS Personal Health Dashboard for account-specific events

## Phase 2 — Remediate

MUST:
- Enable source failover before maintenance if not configured: `aws mediaconnect update-flow --flow-arn <flow-arn> --source-failover-config FailoverMode=FAILOVER,RecoveryWindow=200,State=ENABLED`
- Ensure both sources are in different AZs from the maintenance target
- Notify downstream consumers of potential brief interruptions

SHOULD:
- Test failover before the maintenance window
- Prepare rollback plan in case maintenance causes extended issues
- Monitor flow metrics during the maintenance window

MAY:
- Schedule non-critical flow maintenance during the same window
- Create a temporary backup flow in a different AZ

## Safety Ratings

```
safety_ratings:
  - "health describe-events: GREEN — read-only health event query"
  - "describe-flow: GREEN — read-only flow inspection"
  - "update-flow (failover config): YELLOW — modifies failover behavior"
  - "start-flow / stop-flow: RED — affects live video delivery"
```

## Escalation Conditions

- Maintenance event causes flow outage exceeding the announced duration
- Source failover does not activate during maintenance
- Multiple flows in the same AZ are affected simultaneously
- Post-maintenance flow does not return to ACTIVE state

## Data Sensitivity

- `aws health describe-events`: LOW — returns event metadata
- `aws mediaconnect describe-flow`: MEDIUM — contains flow configuration details
- `aws mediaconnect update-flow`: MEDIUM — modifies flow configuration
- Maintenance event details: LOW — publicly available service health information

## Prohibited Actions

- NEVER ignore scheduled maintenance events for production flows
- NEVER assume maintenance will not affect your flows — always prepare
- NEVER disable failover during a maintenance window
- NEVER make major flow configuration changes during a maintenance window

## Phase 3 — Rollback

1. If flow does not recover after maintenance, restart: `aws mediaconnect stop-flow --flow-arn <flow-arn>` then `aws mediaconnect start-flow --flow-arn <flow-arn>`
2. If failover was enabled for maintenance, decide whether to keep it (recommended)
3. If a temporary backup flow was created, delete after maintenance: `aws mediaconnect delete-flow --flow-arn <backup-flow-arn>`
4. Verify all outputs are delivering post-maintenance
5. Document the maintenance impact and update the disaster recovery plan

## Common Issues

- symptoms: "Flow shows brief interruption during maintenance window"
  diagnosis: "Expected behavior during infrastructure maintenance."
  resolution: "Enable source failover with dual-AZ sources to maintain continuity during maintenance."

- symptoms: "Flow does not recover after maintenance completes"
  diagnosis: "Flow may be stuck in a transitional state."
  resolution: "Stop and restart the flow. If still stuck, contact AWS Support with the flow ARN and maintenance event ID."

- symptoms: "Maintenance event not visible in Health Dashboard"
  diagnosis: "Event may be account-specific or region-specific."
  resolution: "Check AWS Personal Health Dashboard (not the public status page). Use aws health describe-events with appropriate filters."

## Output Format

```yaml
root_cause: "maintenance — <specific_cause>"
evidence:
  - type: health_event
    content: "<AWS Health event details>"
severity: MEDIUM
mitigation:
  immediate: "Enable failover and monitor during maintenance"
  long_term: "Implement dual-AZ source failover for all production flows"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
