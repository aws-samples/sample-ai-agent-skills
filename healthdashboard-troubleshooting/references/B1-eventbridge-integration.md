---
title: "B1 — EventBridge Integration"
description: "Diagnose Health Dashboard EventBridge integration issues"
status: active
severity: MEDIUM
triggers:
  - "health EventBridge not working"
  - "health events not in EventBridge"
  - "health notification rule"
  - "EventBridge health integration"
owner: devops-agent
objective: "Resolve Health Dashboard EventBridge integration issues"
context: "Health events are published to EventBridge with source 'aws.health'. Global events publish in us-east-1. Regional events publish in the affected region. Rules must match the correct event pattern and be in the right region."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rules for health events: `aws events list-rules --region us-east-1`
- Verify rule event pattern: `aws events describe-rule --name <rule-name> --region us-east-1`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name> --region us-east-1`
- Verify rule is enabled and in the correct region

SHOULD:
- Check if rules exist in affected regions (not just us-east-1)
- Verify target permissions (SNS, Lambda, SQS)
- Check CloudWatch metrics for rule invocations: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name Invocations --dimensions Name=RuleName,Value=<rule> --start-time <iso> --end-time <iso> --period 3600 --statistics Sum --region us-east-1`

MAY:
- Check dead-letter queue for failed deliveries
- Review CloudTrail for EventBridge API events

## Phase 2 — Remediate

MUST:
- Create EventBridge rule with correct pattern:
  ```json
  {"source": ["aws.health"], "detail-type": ["AWS Health Event"]}
  ```
- Create rules in us-east-1 for global events AND in monitored regions for regional events
- Configure target with appropriate permissions

SHOULD:
- Filter by event category if needed: add `"detail": {"eventTypeCategory": ["issue"]}` to pattern
- Set up dead-letter queue for failed deliveries
- Test with a known health event

MAY:
- Forward events cross-region using EventBridge rules
- Implement custom event processing with Lambda

## Common Issues

- symptoms: "No health events received in EventBridge"
  diagnosis: "Rule in wrong region or event pattern incorrect."
  resolution: "Create rules in us-east-1 AND affected regions. Use source aws.health."

- symptoms: "Only some health events received"
  diagnosis: "Rule only in us-east-1 (misses regional events) or pattern too restrictive."
  resolution: "Create rules in all monitored regions. Broaden event pattern."

- symptoms: "Target not invoked despite events"
  diagnosis: "Target permissions missing."
  resolution: "Add resource-based policy allowing events.amazonaws.com to invoke target."

## Output Format

```yaml
root_cause: "eventbridge_integration — <specific_cause>"
evidence:
  - type: rule_config
    content: "<EventBridge rule and event pattern>"
  - type: target_config
    content: "<target configuration and permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule pattern, region, or target permissions"
  long_term: "Implement multi-region event capture and processing"
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
