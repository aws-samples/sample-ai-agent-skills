---
title: "E2 — Event Pattern Matching"
description: "Diagnose event pattern matching failures causing missed or incorrect notifications"
status: active
severity: MEDIUM
triggers: ["event not matched", "pattern not working", "wrong notifications", "event filter issue", "EventBridge pattern"]
owner: devops-agent
objective: "Resolve event pattern matching issues for notification delivery"
context: "Event patterns filter which events trigger notifications. Patterns use EventBridge syntax. Non-matching events are silently dropped — no error is generated. Pattern testing is essential."
---

## Phase 1 — Triage

MUST:
- Review the event pattern configuration in the Chatbot channel settings
- Check EventBridge rules if events are routed through EventBridge: `aws events list-rules --name-prefix <prefix>`
- Verify events are being published to the expected source

SHOULD:
- Get rule details: `aws events describe-rule --name <rule-name>`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Test the pattern against a sample event: `aws events test-event-pattern --event-pattern <pattern> --event <sample-event>`

MAY:
- Review CloudWatch metrics for EventBridge rule invocations
- Check EventBridge dead-letter queue for failed deliveries

## Phase 2 — Remediate

MUST:
- Fix event pattern syntax to match desired events
- Test patterns with sample events before deploying
- Verify the event source publishes events in the expected format

SHOULD:
- Use specific event patterns rather than broad matches
- Document event patterns and their intended matches
- Set up monitoring for pattern match rates

MAY:
- Implement EventBridge input transformers for notification formatting
- Create separate rules for different event types
- Use EventBridge Archive and Replay for testing

## Safety Ratings

```
safety_ratings:
  - "events list-rules: GREEN — read-only rule listing"
  - "events describe-rule: GREEN — read-only rule details"
  - "events test-event-pattern: GREEN — read-only pattern testing"
  - "events put-rule: YELLOW — creates or modifies EventBridge rule"
```

## Escalation Conditions

- Critical event patterns not matching production events
- Pattern changes affect multiple notification channels
- Event format changes from source break existing patterns
- Complex patterns required that exceed EventBridge capabilities

## Data Sensitivity

- `aws events list-rules`: LOW — returns rule names and metadata
- `aws events describe-rule`: LOW — returns rule configuration
- `aws events test-event-pattern`: LOW — returns match/no-match result
- Event content: MEDIUM — may contain AWS resource details

## Prohibited Actions

- NEVER use overly broad patterns that match unintended events
- NEVER modify production event patterns without testing with sample events
- NEVER delete EventBridge rules without understanding downstream impact
- NEVER ignore pattern match failures — they indicate configuration issues

## Phase 3 — Rollback

1. If an event pattern was changed incorrectly, restore previous pattern
2. If an EventBridge rule was modified, revert to previous configuration
3. Verify notification delivery resumes with the restored pattern
4. Check that no events were lost during the pattern change
5. Document the pattern change and rollback

## Common Issues

- symptoms: "Events published but no notifications delivered"
  diagnosis: "The event pattern does not match the actual event structure."
  resolution: "Use test-event-pattern to verify. Check event field names, values, and nesting match the pattern."

- symptoms: "Too many notifications — pattern matches unintended events"
  diagnosis: "The event pattern is too broad and matches more events than intended."
  resolution: "Add more specific fields to the pattern. Use detail-type and source fields to narrow matches."

- symptoms: "Pattern worked before but stopped matching"
  diagnosis: "The event source changed its event format or added new fields."
  resolution: "Review current event format from the source. Update the pattern to match the new format."

## Output Format

```yaml
root_cause: "event_pattern — <specific_cause>"
evidence:
  - type: pattern_error
    content: "<pattern and event matching details>"
severity: MEDIUM
mitigation:
  immediate: "Fix event pattern"
  long_term: "Implement pattern testing and monitoring"
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
