---
name: sns-advanced-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot advanced Amazon SNS problems
  by analyzing delivery failures, SMS sending, mobile push notifications,
  subscription filtering, FIFO topics, cross-account access, and DLQ
  configurations using structured runbooks.
  Activate when: delivery failures, delivery status logging issues, DLQ
  configuration, SMS sending failures, spending limits, origination numbers,
  platform application issues, push notification failures, subscription
  filter policies, attribute-based filtering, FIFO topic issues,
  deduplication problems, cross-account subscriptions, access policy
  issues, or the user says something is wrong with SNS without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SNS, SQS, Lambda, CloudWatch,
  CloudTrail, IAM, and optionally Pinpoint permissions.
---

# SNS Advanced Diagnostics

## When to use

Any advanced Amazon SNS investigation where the console alone is insufficient — delivery failure analysis, SMS troubleshooting, mobile push debugging, filter policy configuration, FIFO topic issues, cross-account access, or DLQ management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws sns list-topics
aws sns get-topic-attributes --topic-arn <topic-arn>
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfMessagesPublished --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws sns get-subscription-attributes --subscription-arn <sub-arn>
aws sns get-sms-attributes
aws sns list-platform-applications
aws sns get-platform-application-attributes --platform-application-arn <app-arn>
aws sns check-if-phone-number-is-opted-out --phone-number <number>
aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages
```

### Step 3 — Detailed investigation

```
aws logs filter-log-events --log-group-name sns/<region>/<account-id>/<topic-name> --start-time <epoch-ms> --end-time <epoch-ms>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=sns.amazonaws.com --max-results 10
aws sns list-phone-numbers-opted-out
aws sns list-origination-numbers
```

Read `references/guardrails.md` before concluding on any SNS issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-topic-attributes` | Topic configuration and policy |
| `get-subscription-attributes` | Subscription details and filter |
| `get-sms-attributes` | SMS sending configuration |
| `list-platform-applications` | Mobile push platforms |
| `check-if-phone-number-is-opted-out` | SMS opt-out status |
| `list-origination-numbers` | SMS origination numbers |
| `get-data-protection-policy` | Data protection settings |
| `cloudwatch get-metric-statistics` | Delivery metrics |
| `logs filter-log-events` | Delivery status logs |

## Gotchas: SNS

- Delivery status logging must be explicitly enabled per protocol (HTTP, Lambda, SQS, SMS, Application). It is NOT on by default.
- SMS spending limit: AWS accounts start in the SMS sandbox with a $1.00 monthly limit. Production access requires a spending limit increase request.
- SMS opt-out: Recipients can opt out by replying STOP. Opted-out numbers silently fail delivery. Check opt-out list before diagnosing SMS failures.
- Filter policies: By default, filter policies match against message attributes. MessageBody-based filtering requires setting FilterPolicyScope to MessageBody.
- FIFO topics: Require .fifo suffix. Messages are delivered in order within a message group. Deduplication uses MessageDeduplicationId or content-based deduplication.
- Cross-account: Topic policy must explicitly allow the other account to subscribe or publish. The subscription must be confirmed by the endpoint owner.
- DLQ: Dead-letter queues for SNS are configured per subscription, not per topic. The DLQ must be an SQS queue in the same Region.
- Mobile push: Platform applications require platform-specific credentials (APNs certificate/key for iOS, FCM API key for Android).
- Message size: Standard topics support up to 256 KB. FIFO topics also support 256 KB. For larger payloads, use S3 with a reference in the message.
- Retry policy: SNS has built-in retry policies per protocol. HTTP/S retries for up to 23 days with exponential backoff. Lambda retries 3 times. SQS has no retries (immediate delivery).

## Anti-hallucination rules

1. Always check delivery status logs before concluding on delivery failures — they contain the specific error.
2. SMS sandbox limits sending to verified numbers only — do not assume production SMS access.
3. Filter policies match message attributes by default, NOT message body — verify FilterPolicyScope.
4. FIFO topics require .fifo suffix — standard topics cannot be converted to FIFO.
5. DLQs are per-subscription, not per-topic — each subscription needs its own DLQ configuration.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Delivery | A1-A3 | Delivery failures, delivery status logging, DLQ |
| B — SMS | B1-B3 | SMS sending failures, spending limits, origination numbers |
| C — Mobile Push | C1-C2 | Platform application, push notification failures |
| D — Filtering | D1-D2 | Subscription filter policies, attribute-based |
| E — FIFO | E1-E2 | FIFO topics, deduplication |
| F — Cross-Account | F1-F2 | Cross-account subscriptions, access policy |
| Z — Catch-All | Z1 | General troubleshooting |
