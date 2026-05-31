# SNS Advanced Diagnostics Skill

Agent skill for investigating and troubleshooting advanced Amazon SNS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon SNS when the console alone isn't enough — delivery failure analysis, SMS sending issues, mobile push notifications, subscription filter policies, FIFO topics, cross-account access, and dead-letter queue management.

### Activate When

- Message delivery failures
- Delivery status logging not working
- Dead-letter queue configuration issues
- SMS sending failures
- SMS spending limit problems
- Origination number issues
- Mobile push platform application errors
- Push notification delivery failures
- Subscription filter policy not matching
- Attribute-based filtering issues
- FIFO topic ordering problems
- Message deduplication issues
- Cross-account subscription failures
- Topic access policy problems

---

## Skill Structure

```
sns-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-delivery-failures.md
    ├── A2-delivery-status-logging.md
    ├── A3-dlq.md
    ├── B1-sms-sending-failures.md
    ├── B2-spending-limits.md
    ├── B3-origination-numbers.md
    ├── C1-platform-application.md
    ├── C2-push-notification-failures.md
    ├── D1-filter-policies.md
    ├── D2-attribute-based-filtering.md
    ├── E1-fifo-topics.md
    ├── E2-deduplication.md
    ├── F1-cross-account-subscriptions.md
    ├── F2-access-policy.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Delivery** | A1–A3 | Delivery failures, delivery status logging, DLQ |
| **B — SMS** | B1–B3 | SMS sending failures, spending limits, origination numbers |
| **C — Mobile Push** | C1–C2 | Platform application, push notification failures |
| **D — Filtering** | D1–D2 | Subscription filter policies, attribute-based |
| **E — FIFO** | E1–E2 | FIFO topics, deduplication |
| **F — Cross-Account** | F1–F2 | Cross-account subscriptions, access policy |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering delivery status logging, SMS sandbox, SMS opt-out, filter policy scope, FIFO topic naming, DLQ per-subscription, mobile push credentials, retry policies, message size limits, cross-account confirmation, raw message delivery, and subscription confirmation.

---

## Investigation Workflow

1. **Triage** — Check topic attributes, subscription list, delivery metrics
2. **Deep Dive** — Examine subscription attributes, SMS settings, platform apps, DLQ
3. **Detailed** — Inspect delivery status logs, CloudTrail events, opt-out lists

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `sns:*`, `sqs:*`, `logs:FilterLogEvents`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Delivery status logging enabled (recommended)

---

## Usage Examples

```
# Check topic configuration
aws sns get-topic-attributes --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic

# List subscriptions
aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic

# Check SMS settings
aws sns get-sms-attributes

# Check delivery failures
aws cloudwatch get-metric-statistics --namespace AWS/SNS \
  --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=my-topic \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 3600 --statistics Sum

# Check delivery status logs
aws logs filter-log-events --log-group-name sns/us-east-1/123456789012/my-topic
```

---

## License

MIT-0
