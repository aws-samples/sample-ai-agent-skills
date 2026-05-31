---
title: "E1 — EventBridge Rule Matching"
description: "Diagnose and resolve Amazon EventBridge rule pattern matching failures"
status: active
severity: HIGH
triggers:
  - "EventBridge rule not matching"
  - "event pattern"
  - "rule not triggering"
  - "events not delivered"
  - "event bus"
owner: devops-agent
objective: "Resolve EventBridge rule matching issues and ensure events are properly routed to targets"
context: "EventBridge rules use event patterns to match incoming events on an event bus. Events that match a rule's pattern are routed to the rule's targets. Pattern matching supports exact match, prefix, suffix, numeric comparison, exists, and anything-but operators. Issues arise from incorrect pattern syntax, wrong event bus, event structure mismatches, or disabled rules."
---

## Phase 1 — Triage

MUST:
- Check rule status: `aws events describe-rule --name <rule-name> --event-bus-name <bus-name>`
- Review the event pattern: `aws events describe-rule --name <rule-name> --event-bus-name <bus-name> --query 'EventPattern'`
- Verify the rule is on the correct event bus
- Check CloudWatch metrics for the rule: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name MatchedEvents --dimensions Name=RuleName,Value=<rule-name> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Verify events are being published to the bus: check TriggeredRules metric for the bus

SHOULD:
- Test the event pattern using the EventBridge sandbox: `aws events test-event-pattern --event-pattern '<pattern>' --event '<event-json>'`
- Verify the event source and detail-type match the pattern
- Check if the rule has targets configured: `aws events list-targets-by-rule --rule <rule-name> --event-bus-name <bus-name>`

MAY:
- Enable CloudWatch Logs for the event bus to capture all events
- Check if the event bus has a resource policy that might block events
- Review if the event schema matches the expected structure

## Phase 2 — Remediate

MUST:
- If rule is disabled: enable it: `aws events enable-rule --name <rule-name> --event-bus-name <bus-name>`
- If pattern is incorrect: update the event pattern: `aws events put-rule --name <rule-name> --event-bus-name <bus-name> --event-pattern '<corrected-pattern>'`
- If events on wrong bus: update the publisher to send to the correct bus, or create a rule on the correct bus
- If no targets: add targets: `aws events put-targets --rule <rule-name> --event-bus-name <bus-name> --targets '<targets-json>'`

SHOULD:
- Test the corrected pattern with the sandbox before deploying
- Verify MatchedEvents metric increases after the fix
- Check that targets are successfully invoked after matching

MAY:
- Implement event archiving for replay and debugging
- Set up CloudWatch alarms for MatchedEvents dropping to 0
- Document the event pattern and expected event structure

## Common Issues

- symptoms: "Rule exists but MatchedEvents is 0"
  diagnosis: "Event pattern does not match incoming events, or events are on a different bus."
  resolution: "Test the pattern with actual events using test-event-pattern. Verify the event bus."

- symptoms: "Rule matches some events but not others"
  diagnosis: "Event structure varies — some events have fields the pattern requires and others don't."
  resolution: "Use the 'exists' operator for optional fields. Review event schema variations."

- symptoms: "Pattern with nested fields doesn't match"
  diagnosis: "Event pattern must match the exact JSON structure including nesting."
  resolution: "Verify the pattern matches the event's detail structure. Use the correct nesting in the pattern."

- symptoms: "Custom event bus rule not triggering"
  diagnosis: "Events being sent to the default bus instead of the custom bus."
  resolution: "Verify the PutEvents call specifies the correct EventBusName."

## Output Format

```yaml
root_cause: "EventBridge rule matching failure — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule status and event pattern>"
  - type: matched_events_metric
    content: "<MatchedEvents CloudWatch metric>"
severity: HIGH
mitigation:
  immediate: "Fix event pattern or enable the rule"
  long_term: "Implement event pattern testing and monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/list/test): GREEN — read-only"
  - "Phase 2 rule pattern changes: YELLOW — affects event routing"
  - "Phase 2 enabling/disabling rules: YELLOW — starts/stops event processing"

## Escalation Conditions

escalation_conditions:
  - "Critical events not being routed to targets (alerts, workflows)"
  - "Rule pattern change could affect multiple targets"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves cross-account event delivery"
  - "Event bus resource policy changes are needed"

## Data Sensitivity

data_sensitivity:
  - command: "describe-rule"
    sensitivity: LOW
    contains: "Rule configuration and event pattern"
  - command: "event bus logs"
    sensitivity: HIGH
    contains: "Full event payloads including business data"
  - command: "list-targets-by-rule"
    sensitivity: MEDIUM
    contains: "Target ARNs and configuration"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete EventBridge rules without verifying no critical workflows depend on them"
  - "NEVER use overly broad patterns (e.g., match all events) in production"
  - "NEVER disable rules during active incident response without a replacement"
  - "NEVER modify event bus resource policies without understanding cross-account implications"

## Phase 3 — Rollback

- Before changing event patterns, save the current pattern: `aws events describe-rule --name <rule-name>`
- If pattern changes cause incorrect routing, restore the previous pattern
- If a rule was enabled/disabled incorrectly, toggle it back
- If targets were modified, restore the previous target configuration
- Monitor MatchedEvents and target invocations after any changes
