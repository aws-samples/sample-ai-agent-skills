---
title: "B2 — AWS Chatbot EventBridge Integration Issues"
description: "Diagnose EventBridge event delivery to Chatbot channels"
status: active
severity: MEDIUM
triggers:
  - "EventBridge"
  - "event not appearing"
  - "event rule"
  - "custom event"
owner: devops-agent
objective: "Identify and resolve EventBridge integration issues with AWS Chatbot"
context: "Chatbot can receive events via EventBridge rules that target SNS topics. Not all EventBridge events are supported. Custom events require specific patterns and SNS topic routing."
---

## Phase 1 — Triage

MUST:
- List EventBridge rules: `aws events list-rules --query 'Rules[*].{Name:Name,State:State,EventPattern:EventPattern}'`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify target is the SNS topic configured in Chatbot channel
- Check if rule is ENABLED
- Verify event pattern matches expected events

SHOULD:
- Check EventBridge metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name Invocations --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify SNS topic is subscribed to the Chatbot channel
- Check for failed invocations: FailedInvocations metric

MAY:
- Test with a custom event: `aws events put-events --entries '[{"Source":"test","DetailType":"test","Detail":"{}"}]'`
- Check EventBridge dead-letter queue if configured

## Phase 2 — Remediate

MUST:
- Create EventBridge rule with correct event pattern
- Set rule target to the SNS topic configured in the Chatbot channel
- Enable the rule
- Ensure SNS topic is in the Chatbot channel configuration

SHOULD:
- Use specific event patterns to avoid noise
- Test event delivery end-to-end
- Monitor rule invocation metrics

MAY:
- Set up dead-letter queue for failed deliveries
- Create multiple rules for different event types

## Common Issues

- symptoms: "EventBridge rule exists but events not in Chatbot"
  diagnosis: "Rule target is not the SNS topic configured in the Chatbot channel."
  resolution: "Update rule target to the correct SNS topic ARN."

- symptoms: "Rule targeting correct SNS topic but no notifications"
  diagnosis: "Event pattern doesn't match the actual events being generated."
  resolution: "Review and fix the event pattern to match expected events."

- symptoms: "Events delivered but Chatbot doesn't format them"
  diagnosis: "Event type not supported by Chatbot's notification formatter."
  resolution: "Use a supported event source or accept generic event formatting."

## Output Format

```yaml
root_cause: "eventbridge_issue — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule state, pattern, and targets>"
  - type: metrics
    content: "<invocation and failure metrics>"
  - type: sns_config
    content: "<SNS topic and Chatbot channel link>"
severity: MEDIUM
mitigation:
  immediate: "Fix event pattern or rule target"
  long_term: "Implement event delivery monitoring and alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create EventBridge rule with correct event pattern | GREEN |
| Set rule target to the SNS topic configured in the Chatbot channel | GREEN |
| Enable the rule | GREEN |
| Ensure SNS topic is in the Chatbot channel configuration | GREEN |
| Use specific event patterns to avoid noise | GREEN |
| Test event delivery end-to-end | GREEN |
| Monitor rule invocation metrics | GREEN |
| Set up dead-letter queue for failed deliveries | GREEN |
| Create multiple rules for different event types | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- EventBridge event patterns and payloads may contain operational data

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If EventBridge rules were created, delete them: `aws events delete-rule --name <rule-name>` (remove targets first)
2. If rule targets were changed, restore the previous target configuration
3. If event patterns were modified, revert to the previous pattern
4. Verify rollback by confirming `aws events list-rules` shows expected rules

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
