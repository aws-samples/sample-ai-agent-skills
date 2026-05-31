---
title: "D2 — SNS Filtering"
description: "Diagnose and resolve SNS subscription filter policy issues"
status: active
severity: MEDIUM
triggers:
  - "filter policy"
  - "message filtering"
  - "SNS filter"
  - "message attributes"
  - "FilterPolicyScope"
owner: devops-agent
objective: "Resolve SNS filter policy configuration issues and ensure correct message routing"
context: "SNS subscription filter policies allow subscribers to receive only messages matching specific criteria. Filters can be applied to message attributes (default) or message body (FilterPolicyScope=MessageBody). Filter policies use JSON-based matching with operators like exact match, prefix, numeric range, and exists. Issues arise from incorrect filter syntax, attribute name mismatches, or scope misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check the subscription filter policy: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.FilterPolicy'`
- Check the filter policy scope: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.FilterPolicyScope'`
- Verify the publisher is setting message attributes that match the filter: review publisher code or logs
- Test by publishing a message with known attributes: `aws sns publish --topic-arn <topic-arn> --message "test" --message-attributes '{"key":{"DataType":"String","StringValue":"value"}}'`
- Check CloudWatch NumberOfNotificationsFilteredOut metric

SHOULD:
- Verify the filter policy JSON syntax is valid
- Check if the attribute data types match (String, Number, String.Array)
- Review if the filter uses the correct operators (exact, prefix, anything-but, numeric)

MAY:
- Compare the filter policy against sample messages to verify matching logic
- Check if multiple subscriptions have conflicting or overlapping filters
- Review the SNS filter policy documentation for supported operators

## Phase 2 — Remediate

MUST:
- If filter syntax is invalid: fix the JSON filter policy: `aws sns set-subscription-attributes --subscription-arn <sub-arn> --attribute-name FilterPolicy --attribute-value '<valid-json>'`
- If attribute names don't match: update the filter or publisher to use consistent attribute names
- If scope is wrong: change FilterPolicyScope: `aws sns set-subscription-attributes --subscription-arn <sub-arn> --attribute-name FilterPolicyScope --attribute-value MessageBody`
- If filter is too restrictive: broaden the filter criteria or remove it

SHOULD:
- Test the updated filter with a sample message before relying on it in production
- Document the filter policy and the expected message attribute format
- Verify NumberOfNotificationsFilteredOut decreases after the fix

MAY:
- Implement a catch-all subscription (no filter) for debugging
- Set up monitoring for filtered-out message rates
- Create a filter policy testing tool for the team

## Common Issues

- symptoms: "Subscriber receives no messages despite messages being published"
  diagnosis: "Filter policy does not match any published message attributes."
  resolution: "Review the filter policy and compare with actual message attributes. Fix mismatches."

- symptoms: "Subscriber receives all messages despite having a filter"
  diagnosis: "Filter policy is empty {} or matches all messages."
  resolution: "Update the filter policy with the correct matching criteria."

- symptoms: "Filter works for some messages but not others"
  diagnosis: "Some messages have the required attributes and others don't. Messages without the filtered attribute are rejected."
  resolution: "Ensure all messages include the required attributes, or use the 'exists' operator for optional attributes."

- symptoms: "Filter policy update returns validation error"
  diagnosis: "Invalid JSON syntax or unsupported filter operator."
  resolution: "Validate the JSON. Use supported operators: exact match, prefix, anything-but, numeric, exists."

## Output Format

```yaml
root_cause: "SNS filtering issue — <specific_cause>"
evidence:
  - type: filter_policy
    content: "<current filter policy>"
  - type: message_attributes
    content: "<sample message attributes from publisher>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter policy syntax or attribute matching"
  long_term: "Document filter policies and implement filter testing"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-subscription-attributes): GREEN — read-only"
  - "Phase 2 filter policy changes: YELLOW — affects which messages subscribers receive"
  - "Phase 2 removing filter policy: YELLOW — subscriber will receive all messages"

## Escalation Conditions

escalation_conditions:
  - "Filter policy change affects production message routing"
  - "Removing a filter would expose a subscriber to messages it should not receive"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Filter policy change requires coordinated publisher and subscriber updates"
  - "Issue involves compliance-required message routing"

## Data Sensitivity

data_sensitivity:
  - command: "get-subscription-attributes"
    sensitivity: LOW
    contains: "Filter policy configuration"
  - command: "publish test message"
    sensitivity: MEDIUM
    contains: "Test message content and attributes"
  - command: "message attributes"
    sensitivity: MEDIUM
    contains: "Message metadata that may reveal business logic"

## Prohibited Actions

prohibited_actions:
  - "NEVER remove a filter policy without understanding what messages the subscriber will receive"
  - "NEVER publish production data as test messages for filter debugging"
  - "NEVER modify filter policies on compliance-critical subscriptions without review"
  - "NEVER use overly broad filters that bypass intended message routing"

## Phase 3 — Rollback

- Before changing filter policies, save the current policy: `aws sns get-subscription-attributes --subscription-arn <sub-arn>`
- If filter changes cause wrong messages to be delivered, restore the previous filter
- If filter removal floods the subscriber, re-apply the filter immediately
- If FilterPolicyScope was changed, revert if it breaks message matching
- Monitor message delivery and filtering metrics after any changes
