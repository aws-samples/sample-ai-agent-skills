---
name: cloudtrail-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS CloudTrail problems
  by analyzing trail creation failures, log delivery issues, S3 bucket
  configuration, missing events, data events, management events,
  organization trails, delegated admin, CloudTrail Lake queries, event
  data stores, Insights events, anomaly detection, CloudWatch Logs
  integration, EventBridge rules, log file validation, encryption,
  data event costs, and Lake pricing. Activate when: trail not
  delivering logs, events missing from CloudTrail, data events not
  captured, organization trail issues, CloudTrail Lake query failures,
  Insights not detecting anomalies, CloudWatch Logs not receiving
  events, log validation failures, encryption errors, unexpected
  CloudTrail costs, or the user says something is wrong with
  CloudTrail without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with CloudTrail, S3, CloudWatch Logs,
  EventBridge, KMS, IAM, and optionally Organizations permissions.
---

# AWS CloudTrail Diagnostics

## When to use

Any AWS CloudTrail investigation where the console alone is insufficient — trail configuration, log delivery, missing events, organization trails, CloudTrail Lake, Insights, integrations, security, or cost management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name <trail-name>
aws cloudtrail get-event-selectors --trail-name <trail-name>
aws cloudtrail lookup-events --max-results 10
```

### Step 2 — Domain deep dive

```
aws cloudtrail get-insight-selectors --trail-name <trail-name>
aws cloudtrail list-event-data-stores
aws cloudtrail get-trail --name <trail-name>
aws s3api get-bucket-policy --bucket <trail-bucket>
```

### Step 3 — Detailed investigation

```
aws cloudtrail list-queries --event-data-store <eds-arn>
aws cloudtrail get-query-results --event-data-store <eds-arn> --query-id <query-id>
aws logs describe-log-groups --log-group-name-prefix aws-cloudtrail
aws kms describe-key --key-id <key-id>
```

Read `references/cloudtrail-guardrails.md` before concluding on any CloudTrail issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-trails` | List all trails and their configuration |
| `get-trail-status` | Check trail delivery status and errors |
| `get-event-selectors` | Check management/data event configuration |
| `lookup-events` | Search for specific events |
| `get-insight-selectors` | Check Insights configuration |
| `list-event-data-stores` | List CloudTrail Lake data stores |
| `get-trail` | Get detailed trail configuration |
| S3 bucket policy | Verify CloudTrail can write to the bucket |

## Gotchas: AWS CloudTrail

- Management events are logged by default. Data events and Insights events must be explicitly enabled. Data events (S3 object-level, Lambda invocations) generate high volume and cost.
- CloudTrail delivers logs to S3 with ~5-15 minute delay. It is not real-time. For near-real-time, use CloudWatch Logs integration or EventBridge.
- Organization trails log events for all accounts. They must be created from the management account or delegated admin. Member accounts cannot modify organization trails.
- CloudTrail Lake uses SQL queries on event data stores. Event data stores have retention periods and pricing based on data ingested. Lake is separate from S3-based trails.
- Log file validation ensures integrity. When enabled, CloudTrail creates digest files with hashes. Validation can detect tampering. Digest files are delivered to the same S3 bucket.
- S3 bucket policy must allow CloudTrail to write. The bucket policy needs s3:PutObject and s3:GetBucketAcl permissions for the CloudTrail service principal.
- KMS encryption requires key policy updates. The KMS key policy must allow CloudTrail to encrypt and the trail consumers to decrypt. Missing permissions cause delivery failures.
- Data events can be expensive. S3 object-level logging and Lambda invocation logging generate many events. Use advanced event selectors to filter specific buckets or functions.
- Each region needs its own trail or a multi-region trail. A single-region trail only captures events in that region. Multi-region trails capture events in all regions.

## Anti-hallucination rules

1. Always cite specific trail names, event IDs, or API responses as evidence.
2. Management events are free (first copy). Data events are always paid. Never confuse pricing.
3. CloudTrail is not real-time. Never claim immediate event availability.
4. Organization trails are managed centrally. Never suggest member accounts create them.
5. CloudTrail Lake and S3 trails are separate features. Never confuse their APIs.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Trail | A1–A3 | Trail creation, delivery failures, S3 bucket issues |
| B — Events | B1–B3 | Missing events, data events, management events |
| C — Organization | C1–C2 | Organization trail, delegated admin |
| D — Lake | D1–D2 | CloudTrail Lake queries, event data store |
| E — Insights | E1–E2 | Insights events, anomaly detection |
| F — Integration | F1–F2 | CloudWatch Logs, EventBridge |
| G — Security | G1–G2 | Log file validation, encryption |
| H — Cost | H1–H2 | Data event costs, Lake pricing |
| Z — Catch-All | Z1 | General troubleshooting |
