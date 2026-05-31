---
title: "F2 — Journey Activity Failures"
description: "Diagnose individual journey activity failures"
status: active
severity: MEDIUM
triggers:
  - "journey activity failed"
  - "wait activity stuck"
  - "condition not evaluating"
  - "journey message not sent"
owner: devops-agent
objective: "Identify and resolve journey activity failures"
context: "Journey activities include message send, wait, condition split, multivariate split, and holdout. Each can fail independently. Activity failures don't stop the journey but skip affected participants."
---

## Phase 1 — Triage

MUST:
- Get journey activity metrics: `aws pinpoint get-journey-execution-activity-metrics --application-id <app-id> --journey-id <journey-id> --journey-activity-id <activity-id>`
- Check journey configuration for activity details: `aws pinpoint get-journey --application-id <app-id> --journey-id <journey-id>`
- Verify message template exists if activity uses one
- Check channel configuration for message activities

SHOULD:
- Review condition expressions for syntax errors
- Verify wait durations are within limits
- Check if holdout percentage is too high

MAY:
- Compare expected vs actual participant flow
- Review event-based condition triggers

## Phase 2 — Remediate

MUST:
- Fix message template or channel configuration for send activities
- Correct condition expressions
- Ensure wait durations don't exceed journey end date

SHOULD:
- Test each activity type independently
- Use journey simulation before production launch

MAY:
- Implement per-activity monitoring

## Common Issues

- symptoms: "Message activity shows high failure rate"
  diagnosis: "Channel not configured or template rendering error."
  resolution: "Verify channel enabled. Check template variables match endpoint attributes."

- symptoms: "Condition split sends all participants to 'No' branch"
  diagnosis: "Condition expression references non-existent attribute."
  resolution: "Fix attribute name in condition. Verify endpoints have the attribute."

## Output Format

```yaml
root_cause: "journey_activity — <specific_cause>"
evidence:
  - type: activity_metrics
    content: "<activity execution metrics>"
  - type: activity_config
    content: "<activity configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix activity configuration"
  long_term: "Implement journey testing framework"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — journey activities process customer data
- Service Disruption Risk: MEDIUM — activity failures disrupt customer journeys
- Compliance Impact: HIGH — journey activities must comply with messaging regulations

## Escalation Conditions
- Journey activities fail to execute
- Activities send messages to wrong segments
- Activity conditions evaluate incorrectly
- Wait activities cause unexpected delays
- Multi-channel activities fail on specific channels

## Data Sensitivity
- Customer journey data — RESTRICTED
- Activity configurations — INTERNAL
- Condition expressions — INTERNAL
- Channel-specific content — INTERNAL
- Never expose customer journey paths or PII

## Prohibited Actions
- Do NOT modify active journey activities without pausing the journey
- Do NOT bypass activity validation
- Do NOT create activities that violate messaging regulations
- Do NOT ignore activity execution failures
- Do NOT create infinite loops in journey activities
- Do NOT expose customer journey data externally

## Phase 3 — Rollback
- Pause the journey and restore previous activity configurations
- Revert condition expression changes
- Restore channel-specific content
- Verify activity execution after changes
- Document activity issues and journey impact
- Update journey design procedures

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
