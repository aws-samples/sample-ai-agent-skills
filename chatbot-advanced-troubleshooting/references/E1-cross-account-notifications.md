---
title: "E1 — Cross-Account Notifications"
description: "Diagnose cross-account notification delivery failures including SNS topic policies, subscription issues, and account permissions"
status: active
severity: HIGH
triggers: ["cross-account notification failed", "other account notifications not working", "SNS cross-account", "multi-account alerts"]
owner: devops-agent
objective: "Resolve cross-account notification delivery issues in AWS Chatbot"
context: "Cross-account notifications require the SNS topic in the source account to have a resource policy allowing the Chatbot account to subscribe. Both the topic policy and Chatbot configuration must be correct."
---

## Phase 1 — Triage

MUST:
- Check the SNS topic policy in the source account: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Verify the Chatbot configuration references the cross-account topic ARN
- Check subscription status: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`

SHOULD:
- Verify the source account publishes events to the topic
- Check if the subscription is confirmed or pending
- Review the topic access policy for cross-account permissions

MAY:
- Test with a direct SNS publish from the source account
- Check CloudTrail in both accounts for subscription events

## Phase 2 — Remediate

MUST:
- Update the SNS topic resource policy to allow cross-account subscription
- Ensure the Chatbot configuration includes the correct cross-account topic ARN
- Confirm the SNS subscription

SHOULD:
- Use specific account IDs in the topic policy (not wildcard)
- Verify events are published to the topic in the source account
- Test notification delivery after policy changes

MAY:
- Set up EventBridge cross-account event bus for more flexible routing
- Implement centralized notification aggregation

## Safety Ratings

```
safety_ratings:
  - "sns get-topic-attributes: GREEN — read-only topic metadata"
  - "sns list-subscriptions-by-topic: GREEN — read-only subscription listing"
  - "sns set-topic-attributes (policy): YELLOW — modifies topic access policy"
  - "sns subscribe: YELLOW — creates new subscription to topic"
```

## Escalation Conditions

- Cross-account topic policy changes require source account admin approval
- Critical alerts from other accounts not being delivered
- Topic policy modification affects other subscribers
- Organizational policy restricts cross-account SNS access

## Data Sensitivity

- `aws sns get-topic-attributes`: MEDIUM — reveals topic policy and account details
- `aws sns list-subscriptions-by-topic`: LOW — returns subscription endpoints
- SNS topic policy: MEDIUM — contains account IDs and access rules
- Notification content: MEDIUM — contains AWS event details from source account

## Prohibited Actions

- NEVER use wildcard (*) principals in SNS topic policies for cross-account access
- NEVER modify topic policies without source account owner approval
- NEVER subscribe to topics in accounts you do not have authorization to monitor
- NEVER expose source account details in notification responses

## Phase 3 — Rollback

1. If the topic policy was modified incorrectly, restore previous policy in source account
2. If an unwanted subscription was created, unsubscribe: `aws sns unsubscribe --subscription-arn <arn>`
3. If the Chatbot configuration was changed, revert to previous settings
4. Verify other subscribers to the topic are not affected
5. Document the cross-account configuration change

## Common Issues

- symptoms: "Chatbot configuration saved but no notifications from other account"
  diagnosis: "The SNS topic in the source account does not allow cross-account subscription."
  resolution: "Update the SNS topic resource policy to allow sns:Subscribe from the Chatbot account."

- symptoms: "Subscription shows as PendingConfirmation"
  diagnosis: "The cross-account subscription requires confirmation that has not been completed."
  resolution: "Confirm the subscription. For HTTPS endpoints, the confirmation request must be acknowledged."

- symptoms: "Notifications delivered intermittently from other account"
  diagnosis: "The source account event publishing is inconsistent, or SNS delivery retries are failing."
  resolution: "Check event publishing in the source account. Review SNS delivery status logging."

## Output Format

```yaml
root_cause: "cross_account_notification — <specific_cause>"
evidence:
  - type: cross_account_error
    content: "<topic policy and subscription details>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account SNS configuration"
  long_term: "Implement centralized notification architecture"
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
