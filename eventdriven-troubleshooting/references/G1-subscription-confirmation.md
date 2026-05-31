---
title: "G1 — Subscription Confirmation"
description: "Diagnose and resolve SNS subscription confirmation and management issues"
status: active
severity: MEDIUM
triggers:
  - "subscription pending"
  - "PendingConfirmation"
  - "confirm subscription"
  - "subscription not confirmed"
  - "unsubscribe"
owner: devops-agent
objective: "Resolve SNS subscription confirmation issues and ensure proper subscription lifecycle management"
context: "SNS requires subscription confirmation for HTTP/S, email, and email-JSON protocols. SQS, Lambda, and SMS subscriptions in the same account are auto-confirmed. Cross-account SQS subscriptions require confirmation. Pending subscriptions do not receive messages. Confirmation tokens expire after 3 days. Issues arise from missed confirmation messages, incorrect endpoint URLs, email spam filters, or cross-account permission problems."
---

## Phase 1 — Triage

MUST:
- List subscriptions and check status: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>` (look for 'PendingConfirmation' in SubscriptionArn)
- Verify the endpoint is correct (URL, email address, queue ARN)
- Check if the confirmation message was received at the endpoint
- For HTTP/S: verify the endpoint is accessible and returns 2xx: `curl -v <endpoint-url>`
- For email: check spam/junk folders for the confirmation email

SHOULD:
- Check if the subscription was recently created: review CloudTrail for Subscribe API calls
- Verify the topic access policy allows the subscriber to confirm
- For cross-account SQS: check if the queue policy allows SNS confirmation messages

MAY:
- Check if the endpoint has previously confirmed and then unsubscribed
- Review if there are multiple pending subscriptions for the same endpoint
- Check if the confirmation token has expired (3-day window)

## Phase 2 — Remediate

MUST:
- If confirmation expired: delete the pending subscription and resubscribe: `aws sns unsubscribe --subscription-arn <pending-sub-arn>` then `aws sns subscribe --topic-arn <topic-arn> --protocol <protocol> --notification-endpoint <endpoint>`
- If HTTP/S endpoint not receiving: fix the endpoint to accept POST requests and return 2xx for SubscriptionConfirmation messages
- If email not received: check spam filters, verify the email address, and resubscribe
- If cross-account SQS: ensure the queue policy allows sns:SendMessage from the topic

SHOULD:
- For HTTP/S endpoints: implement automatic confirmation by parsing the SubscribeURL from the confirmation message and making a GET request to it
- Verify the subscription transitions to 'Confirmed' after remediation
- Set up monitoring for pending subscriptions

MAY:
- Implement a subscription management dashboard
- Set up alerts for subscriptions that remain pending for more than 1 hour
- Document the subscription confirmation process for each protocol

## Common Issues

- symptoms: "Subscription shows 'PendingConfirmation' indefinitely"
  diagnosis: "Confirmation message was not received or processed at the endpoint."
  resolution: "Verify the endpoint is accessible. Resubscribe to trigger a new confirmation message."

- symptoms: "HTTP/S endpoint receives confirmation but subscription stays pending"
  diagnosis: "Endpoint is not visiting the SubscribeURL to confirm."
  resolution: "Update the endpoint to parse the SubscriptionConfirmation message and GET the SubscribeURL."

- symptoms: "Email subscription confirmation not received"
  diagnosis: "Email blocked by spam filter, incorrect email address, or email service issue."
  resolution: "Check spam folders. Verify the email address. Whitelist the SNS sender. Resubscribe."

- symptoms: "Cross-account SQS subscription pending"
  diagnosis: "SQS queue policy does not allow SNS to send the confirmation message."
  resolution: "Add sns:SendMessage permission for the SNS topic ARN in the SQS queue policy."

## Output Format

```yaml
root_cause: "Subscription confirmation issue — <specific_cause>"
evidence:
  - type: subscription_status
    content: "<subscription list with pending status>"
  - type: endpoint_check
    content: "<endpoint accessibility verification>"
severity: MEDIUM
mitigation:
  immediate: "Fix endpoint, resubscribe, or confirm manually"
  long_term: "Implement automatic confirmation and subscription monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (list/get): GREEN — read-only"
  - "Phase 2 resubscribe: YELLOW — creates new subscription"
  - "Phase 2 unsubscribe pending: GREEN — removes non-functional subscription"

## Escalation Conditions

escalation_conditions:
  - "Critical notification subscriptions cannot be confirmed"
  - "Cross-account subscription requires policy changes in another account"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Multiple subscriptions are pending across different topics"
  - "Endpoint infrastructure changes are needed to support confirmation"

## Data Sensitivity

data_sensitivity:
  - command: "list-subscriptions-by-topic"
    sensitivity: MEDIUM
    contains: "Subscriber endpoints (email addresses, URLs, queue ARNs)"
  - command: "confirmation messages"
    sensitivity: LOW
    contains: "Subscription confirmation tokens and URLs"
  - command: "CloudTrail Subscribe events"
    sensitivity: MEDIUM
    contains: "Subscription details and principal information"

## Prohibited Actions

prohibited_actions:
  - "NEVER confirm subscriptions for endpoints you do not control"
  - "NEVER share subscription confirmation URLs publicly"
  - "NEVER delete confirmed subscriptions to troubleshoot — unsubscribe is permanent"
  - "NEVER subscribe endpoints without the endpoint owner's consent"

## Phase 3 — Rollback

- If a subscription was accidentally unsubscribed, resubscribe and confirm
- If a new subscription causes unwanted message delivery, unsubscribe it
- If endpoint changes break confirmation, revert the endpoint changes
- If queue policy changes for cross-account were too broad, tighten them after confirmation
- Verify message delivery works correctly after subscription changes
