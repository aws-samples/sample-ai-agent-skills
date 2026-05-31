---
name: eventbridge-advanced-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon EventBridge
  advanced features including Scheduler, Pipes, Schema Registry,
  Archive/Replay, cross-account delivery, DLQ configuration, and
  partner integrations by analyzing configurations, CloudWatch metrics,
  and following structured runbooks.
  Activate when: schedule creation failures, cron/rate expression errors,
  missed invocations, pipe creation failures, source-to-target errors,
  enrichment Lambda failures, filtering issues, schema registry problems,
  schema discovery, archive configuration, replay failures, cross-account
  event delivery, event bus policies, DLQ configuration, failed event
  analysis, partner integration issues, SaaS event problems, or the user
  says something is wrong with EventBridge Scheduler, Pipes, or advanced
  features without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with EventBridge (events), Scheduler,
  Pipes, CloudWatch, CloudTrail, SQS, Lambda, IAM, and optionally
  Kinesis, DynamoDB Streams, MSK, and MQ permissions.
---

# EventBridge Advanced Diagnostics (Scheduler + Pipes)

## When to use

Any EventBridge advanced feature investigation where the console alone is insufficient — Scheduler debugging, Pipes source-to-target troubleshooting, Schema Registry issues, Archive/Replay problems, cross-account event delivery, DLQ analysis, or partner integration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws scheduler list-schedules
aws scheduler get-schedule --name <schedule-name>
aws pipes list-pipes
aws pipes describe-pipe --name <pipe-name>
aws events list-event-buses
aws events describe-event-bus --name <bus-name>
aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name Invocations --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name FailedInvocations --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws scheduler list-schedule-groups
aws scheduler get-schedule-group --name <group-name>
aws pipes list-pipes --current-state RUNNING
aws events list-archives
aws events describe-archive --archive-name <archive-name>
aws events list-replays
aws events describe-replay --replay-name <replay-name>
aws schemas list-registries
aws schemas list-schemas --registry-name <registry-name>
aws schemas describe-schema --registry-name <registry-name> --schema-name <schema-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=scheduler.amazonaws.com --max-results 20
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=pipes.amazonaws.com --max-results 20
aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names All
aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 10
aws lambda get-function --function-name <enrichment-function>
aws iam get-role --role-name <execution-role>
```

Read `references/eventbridge-advanced-guardrails.md` before concluding on any EventBridge advanced issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `scheduler get-schedule` | Schedule configuration and status |
| `pipes describe-pipe` | Pipe configuration, source, target, enrichment |
| `events describe-event-bus` | Event bus configuration and policy |
| `events describe-archive` | Archive configuration and status |
| `events describe-replay` | Replay status and progress |
| `schemas describe-schema` | Schema definition and versions |
| `sqs receive-message` | DLQ failed event analysis |
| `lambda get-function` | Enrichment function configuration |
| `CloudWatch Events metrics` | Invocation and failure metrics |
| `CloudTrail` | API call history and errors |

## Gotchas: EventBridge Advanced

- Scheduler is separate from EventBridge rules (different API). EventBridge Scheduler uses the scheduler API namespace, not the events API. Schedules are not rules — they have different configuration, IAM roles, and delivery mechanisms. Scheduler supports one-time and recurring schedules with flexible time windows.
- Pipes connect source directly to target (no rules needed). Pipes provide point-to-point integration between a source and target with optional filtering and enrichment. They do NOT use event buses or rules. Pipes have their own IAM execution role. Pipes are different from EventBridge rules with targets.
- Scheduler supports one-time and recurring schedules. One-time schedules execute once at a specified time and are automatically deleted after execution (configurable). Recurring schedules use cron or rate expressions. Flexible time windows allow execution within a window rather than exact time.
- Pipes sources are limited to specific services. Pipe sources are limited to: SQS, Kinesis Data Streams, DynamoDB Streams, Amazon MSK, and Amazon MQ. You cannot use arbitrary event sources. Each source type has specific configuration options (batch size, starting position, etc.).
- Enrichment is optional Lambda/Step Functions/API Gateway/API Destination. Pipe enrichment transforms events between source and target. It is optional. Enrichment adds latency and cost. Enrichment failures can be handled with retry policies and DLQ. The enrichment function receives the filtered events and returns transformed events.
- Archive replay delivers events to the same event bus. When replaying archived events, they are delivered to the same event bus they were originally sent to. All rules on that bus will process the replayed events. Use event patterns to filter replayed events if needed. Replays include a replay-name header for identification.
- Schema registry auto-discovers from events on the bus. Schema discovery must be enabled on the event bus. It automatically creates schemas from events flowing through the bus. Discovered schemas are stored in the discovered-schemas registry. Schema discovery adds minimal latency but incurs costs per event analyzed.

## Anti-hallucination rules

1. Always cite specific schedule names, pipe names, archive names, or API responses as evidence.
2. Scheduler and EventBridge rules are DIFFERENT services with different APIs. Never confuse them.
3. Pipes do NOT use event buses or rules. Never suggest configuring rules for pipe-based integrations.
4. Pipe sources are limited to SQS, Kinesis, DynamoDB Streams, MSK, and MQ. Never suggest other sources.
5. Archive replay sends events to the SAME event bus. Never suggest replaying to a different bus.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Scheduler | A1-A3 | Creation failures, cron/rate errors, missed invocations |
| B — Pipes | B1-B4 | Creation failures, source-to-target errors, enrichment failures, filtering |
| C — Schema | C1-C2 | Schema registry, schema discovery |
| D — Archive | D1-D2 | Archive configuration, replay failures |
| E — Cross-Account | E1-E2 | Cross-account delivery, event bus policies |
| F — DLQ | F1-F2 | DLQ configuration, failed event analysis |
| G — Partner | G1-G2 | Partner integration, SaaS events |
| Z — Catch-All | Z1 | General troubleshooting |
