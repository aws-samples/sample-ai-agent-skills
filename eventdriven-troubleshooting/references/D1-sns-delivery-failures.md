---
title: "D1 — SNS Delivery Failures"
description: "Diagnose and resolve Amazon SNS message delivery failures to subscribers"
status: active
severity: HIGH
triggers:
  - "SNS delivery failed"
  - "subscription not receiving"
  - "SNS notification failed"
  - "delivery status logging"
  - "NumberOfNotificationsFailed"
owner: devops-agent
objective: "Identify why SNS messages are not being delivered to subscribers and restore delivery"
context: "SNS delivers messages to subscribed endpoints: SQS, Lambda, HTTP/S, email, SMS, and mobile push. Delivery failures can occur due to endpoint unavailability, permission issues, subscription filter policies, or protocol-specific errors. SNS retries failed deliveries based on the delivery policy. Delivery status logging can be enabled for SQS, Lambda, HTTP/S, and mobile push."
---

## Phase 1 — Triage

MUST:
- Check topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify subscription status is 'Confirmed' (not 'PendingConfirmation')
- Check CloudWatch NumberOfNotificationsFailed: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic-name> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Check delivery status logging (if enabled): review CloudWatch Logs for SNS delivery logs
- Verify the topic has the correct access policy: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.Policy'`

SHOULD:
- Check if a subscription filter policy is filtering out messages: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.FilterPolicy'`
- Verify the endpoint is accessible (SQS queue exists, Lambda function exists, HTTP endpoint responds)
- Check if the SQS queue policy allows SNS to send messages
- Review the SNS delivery policy for retry configuration

MAY:
- Enable delivery status logging for the protocol: `aws sns set-topic-attributes --topic-arn <topic-arn> --attribute-name <Protocol>SuccessFeedbackRoleArn --attribute-value <role-arn>`
- Check if the topic is encrypted and the subscriber has KMS decrypt permissions
- Review CloudTrail for Publish API errors

## Phase 2 — Remediate

MUST:
- If subscription not confirmed: resend confirmation or resubscribe the endpoint
- If SQS policy blocks SNS: add SNS publish permission to the SQS queue policy
- If Lambda permission missing: add SNS invoke permission: `aws lambda add-permission --function-name <fn> --statement-id sns --action lambda:InvokeFunction --principal sns.amazonaws.com --source-arn <topic-arn>`
- If filter policy too restrictive: update or remove the filter: `aws sns set-subscription-attributes --subscription-arn <sub-arn> --attribute-name FilterPolicy --attribute-value '{}'`

SHOULD:
- Enable delivery status logging for debugging future failures
- Verify delivery resumes after remediation by publishing a test message
- Configure a DLQ for the subscription to capture failed deliveries

MAY:
- Implement SNS message archiving for compliance
- Set up CloudWatch alarms for NumberOfNotificationsFailed
- Review and optimize the delivery retry policy

## Common Issues

- symptoms: "Messages published but SQS subscriber doesn't receive them"
  diagnosis: "SQS queue policy does not allow sns:SendMessage from the topic."
  resolution: "Add a policy statement allowing the SNS topic ARN to send messages to the queue."

- symptoms: "HTTP/S endpoint not receiving notifications"
  diagnosis: "Endpoint returning non-2xx status, or subscription not confirmed."
  resolution: "Verify the endpoint is healthy and returns 2xx. Confirm the subscription if pending."

- symptoms: "Lambda subscriber not invoked"
  diagnosis: "Lambda function does not have SNS invoke permission, or function is throttled."
  resolution: "Add Lambda resource-based policy for SNS. Check Lambda concurrent execution limits."

- symptoms: "Some messages delivered but not others"
  diagnosis: "Subscription filter policy is filtering out messages that don't match."
  resolution: "Review the filter policy. Ensure message attributes match the filter criteria."

## Output Format

```yaml
root_cause: "SNS delivery failure — <specific_cause>"
evidence:
  - type: subscription_status
    content: "<subscription list and confirmation status>"
  - type: delivery_logs
    content: "<delivery status log entries>"
severity: HIGH
mitigation:
  immediate: "Fix permissions, confirm subscriptions, or adjust filter policies"
  long_term: "Enable delivery status logging and implement DLQ for failed deliveries"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (list/get/describe): GREEN — read-only"
  - "Phase 2 policy changes: YELLOW — affects message delivery permissions"
  - "Phase 2 filter policy changes: YELLOW — affects which messages are delivered"

## Escalation Conditions

escalation_conditions:
  - "Critical notifications not being delivered (alerts, alarms)"
  - "Delivery failures affect multiple subscribers simultaneously"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves cross-account SNS delivery"
  - "Encrypted topic delivery failures require KMS policy changes"

## Data Sensitivity

data_sensitivity:
  - command: "get-topic-attributes"
    sensitivity: MEDIUM
    contains: "Topic policy and configuration"
  - command: "delivery status logs"
    sensitivity: HIGH
    contains: "Message delivery details and endpoint information"
  - command: "list-subscriptions-by-topic"
    sensitivity: MEDIUM
    contains: "Subscriber endpoints (email addresses, URLs, queue ARNs)"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete SNS subscriptions to troubleshoot delivery — unsubscribe is permanent"
  - "NEVER publish test messages with sensitive data to debug delivery"
  - "NEVER modify topic access policy to allow * principal"
  - "NEVER disable encryption on a topic to troubleshoot delivery"

## Phase 3 — Rollback

- Before changing topic or queue policies, save the current policy
- If policy changes break other publishers or subscribers, restore the previous policy
- If filter policy changes cause unwanted message delivery, restore the previous filter
- If Lambda permissions were added incorrectly, remove the permission statement
- Monitor delivery success rates after any changes
