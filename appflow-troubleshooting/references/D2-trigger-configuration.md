---
title: "D2 — Trigger Configuration"
description: "Diagnose Amazon AppFlow trigger configuration issues"
status: active
severity: MEDIUM
triggers: ["trigger not working", "event-driven flow not firing", "trigger configuration error"]
owner: devops-agent
objective: "Resolve trigger configuration issues for on-demand, scheduled, and event-driven flows"
context: "AppFlow supports three trigger types: on-demand, scheduled, and event-driven. Event-driven requires source connector support."
---

## Phase 1 — Triage

MUST:
- Check trigger type in flow config: `aws appflow describe-flow --flow-name <flow-name>`
- For scheduled: verify cron expression and schedule offset
- For event-driven: verify source connector supports events

SHOULD:
- Check if the flow is in ACTIVE state
- Review execution records for trigger-related errors: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 5`

MAY:
- For Salesforce event-driven: verify Platform Events are configured in Salesforce
- Check EventBridge for related rules: `aws events list-rules --name-prefix appflow`

## Phase 2 — Remediate

MUST:
- Fix trigger type configuration to match intended behavior
- For event-driven: ensure source connector and SaaS platform are configured for events
- Verify flow is ACTIVE and not SUSPENDED

SHOULD:
- Test trigger by running an on-demand execution first
- Set up monitoring for trigger failures

MAY:
- Implement fallback scheduled trigger for event-driven flows

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow details"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "events list-rules: GREEN — read-only EventBridge rules"
  - "update-flow (trigger change): YELLOW — modifies flow trigger behavior"
```

## Escalation Conditions

- Event-driven trigger requires SaaS platform configuration changes
- Trigger type change affects data freshness SLAs
- Multiple flows depend on the same event source
- Trigger failures are intermittent and difficult to reproduce

## Data Sensitivity

- `aws appflow describe-flow`: MEDIUM — reveals trigger configuration
- `aws appflow describe-flow-execution-records`: LOW — reveals execution timestamps
- `aws events list-rules`: LOW — reveals EventBridge configuration

## Prohibited Actions

- NEVER change trigger type from event-driven to scheduled without understanding data freshness impact
- NEVER configure event-driven triggers for connectors that do not support them
- NEVER remove schedule offset without understanding its purpose (deduplication window)
- NEVER disable triggers on production flows without a maintenance window

## Phase 3 — Rollback

1. If trigger type was changed, revert to original trigger configuration
2. If event-driven configuration was modified in the SaaS platform, restore original settings
3. Run a manual execution to verify data pipeline continuity
4. Verify the trigger fires correctly after rollback
5. Document trigger configuration and dependencies

## Common Issues

- symptoms: "Event-driven flow never fires despite source changes"
  diagnosis: "Source connector does not support event-driven triggers, or SaaS platform events are not configured."
  resolution: "Verify connector supports events. For Salesforce, configure Platform Events or Change Data Capture."

## Output Format

```yaml
root_cause: "trigger_configuration — <specific_cause>"
evidence:
  - type: trigger_detail
    content: "<trigger config>"
severity: MEDIUM
mitigation:
  immediate: "Fix trigger"
  long_term: "Monitor trigger health"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
