# Event-Driven Architecture Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon SQS, SNS, and EventBridge problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for SQS, SNS, and EventBridge when the console alone isn't enough — message delivery failures, DLQ analysis, FIFO ordering issues, SNS fanout problems, subscription filtering, EventBridge rule pattern matching, target invocation errors, Pipes and archive/replay issues, cross-account event delivery, encryption errors, and integration troubleshooting.

### Activate When

- SQS messages not being received by consumers
- Visibility timeout causing duplicate processing
- DLQ overflow or unexpected DLQ messages
- SQS FIFO ordering or deduplication problems
- SQS throughput or throttling issues
- SNS subscription confirmation failures
- SNS delivery failures to HTTP/S, SQS, Lambda, or email
- SNS filter policy not matching expected messages
- Cross-account SNS publish or subscribe failures
- EventBridge rule not matching events
- EventBridge target invocation failures
- EventBridge scheduled rule not firing
- EventBridge Pipes not processing events
- EventBridge archive/replay issues
- Lambda trigger failures from SQS or SNS
- SQS-SNS fanout delivery problems
- Access policy or encryption errors on SQS/SNS/EventBridge

---

## Skill Structure

```
eventdriven-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-messages-not-received.md
    ├── A2-visibility-timeout.md
    ├── A3-dlq-overflow.md
    ├── A4-message-loss.md
    ├── B1-fifo-ordering.md
    ├── B2-fifo-deduplication.md
    ├── B3-fifo-throughput.md
    ├── C1-sqs-throttling.md
    ├── C2-long-polling.md
    ├── C3-batch-operations.md
    ├── D1-subscription-failures.md
    ├── D2-sns-delivery-failures.md
    ├── D3-sns-filtering.md
    ├── D4-sns-cross-account.md
    ├── E1-topic-policy.md
    ├── E2-sns-encryption.md
    ├── E3-raw-message-delivery.md
    ├── F1-pattern-matching.md
    ├── F2-target-failures.md
    ├── F3-scheduling.md
    ├── F4-eventbridge-cross-account.md
    ├── G1-pipes-issues.md
    ├── G2-archive-replay.md
    ├── G3-schema-registry.md
    ├── H1-lambda-triggers.md
    ├── H2-sqs-sns-fanout.md
    ├── H3-eventbridge-to-sqs-sns.md
    ├── I1-access-policies.md
    ├── I2-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── eventdriven-guardrails.md
    └── eventdriven-hallucination-patterns.yaml
```

---

## Runbook Library (36 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — SQS Delivery** | A1–A4 | Messages not received, visibility timeout, DLQ overflow, message loss |
| **B — SQS FIFO** | B1–B3 | Ordering issues, deduplication, throughput limits |
| **C — SQS Performance** | C1–C3 | Throttling, long polling, batch operations |
| **D — SNS Delivery** | D1–D4 | Subscription failures, delivery failures, filtering, cross-account |
| **E — SNS Configuration** | E1–E3 | Topic policy, encryption, raw message delivery |
| **F — EventBridge Rules** | F1–F4 | Pattern matching, target failures, scheduling, cross-account |
| **G — EventBridge Advanced** | G1–G3 | Pipes issues, archive/replay, schema registry |
| **H — Integration** | H1–H3 | Lambda triggers, SQS-SNS fanout, EventBridge-to-SQS/SNS |
| **I — Security** | I1–I2 | Access policies, encryption (SSE-SQS/SSE-KMS) |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
