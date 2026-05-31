# Amazon EventBridge Advanced Diagnostics Skill (Scheduler + Pipes)

Agent skill for investigating and troubleshooting Amazon EventBridge advanced features including Scheduler, Pipes, Schema Registry, Archive/Replay, cross-account delivery, DLQ configuration, and partner integrations using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for EventBridge advanced features when the console alone isn't enough — Scheduler debugging, Pipes source-to-target troubleshooting, Schema Registry issues, Archive/Replay problems, cross-account event delivery, DLQ analysis, and partner integration issues.

### Activate When

- Schedule creation failures
- Cron or rate expression errors
- Missed schedule invocations
- Pipe creation or configuration failures
- Pipe source-to-target delivery errors
- Enrichment Lambda failures in Pipes
- Pipe filtering not matching expected events
- Schema registry configuration issues
- Schema discovery not working
- Archive configuration problems
- Event replay failures
- Cross-account event delivery issues
- Event bus policy configuration
- DLQ not receiving failed events
- Failed event analysis from DLQ
- Partner event source integration issues
- SaaS event delivery problems

---

## Skill Structure

```
eventbridge-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-schedule-creation-failures.md
    ├── A2-cron-rate-expression-errors.md
    ├── A3-missed-invocations.md
    ├── B1-pipe-creation-failures.md
    ├── B2-source-to-target-errors.md
    ├── B3-enrichment-failures.md
    ├── B4-filtering-issues.md
    ├── C1-schema-registry.md
    ├── C2-schema-discovery.md
    ├── D1-archive-configuration.md
    ├── D2-replay-failures.md
    ├── E1-cross-account-delivery.md
    ├── E2-event-bus-policies.md
    ├── F1-dlq-configuration.md
    ├── F2-failed-event-analysis.md
    ├── G1-partner-integration.md
    ├── G2-saas-events.md
    ├── Z1-general-troubleshooting.md
    ├── eventbridge-advanced-guardrails.md
    └── eventbridge-advanced-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Scheduler** | A1–A3 | Creation failures, cron/rate errors, missed invocations |
| **B — Pipes** | B1–B4 | Creation failures, source-to-target errors, enrichment failures, filtering |
| **C — Schema** | C1–C2 | Schema registry, schema discovery |
| **D — Archive** | D1–D2 | Archive configuration, replay failures |
| **E — Cross-Account** | E1–E2 | Cross-account delivery, event bus policies |
| **F — DLQ** | F1–F2 | DLQ configuration, failed event analysis |
| **G — Partner** | G1–G2 | Partner integration, SaaS events |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/eventbridge-advanced-guardrails.md` covering Scheduler vs rules distinction, Pipes architecture, Pipe source limitations, enrichment behavior, archive replay destination, schema discovery costs, cross-account event bus policies, DLQ configuration requirements, one-time schedule deletion, flexible time windows, Pipe batching behavior, and partner event source lifecycle.

---

## Investigation Workflow

1. **Triage** — List schedules/pipes, check CloudWatch metrics, identify the specific feature and configuration
2. **Deep Dive** — Examine schedule groups, pipe details, archives, replays, schemas, registries
3. **Detailed** — Inspect CloudTrail events, DLQ messages, enrichment functions, IAM roles

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `scheduler:*`, `pipes:*`, `events:*`, `schemas:*`, `sqs:*`, `lambda:GetFunction`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- CloudWatch metrics enabled
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# List EventBridge Scheduler schedules
aws scheduler list-schedules

# Describe a specific schedule
aws scheduler get-schedule --name my-schedule

# List EventBridge Pipes
aws pipes list-pipes

# Describe a specific pipe
aws pipes describe-pipe --name my-pipe

# List archives
aws events list-archives

# Check DLQ for failed events
aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 10
```

---

## License

MIT-0
