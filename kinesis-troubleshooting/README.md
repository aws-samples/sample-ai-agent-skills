# Kinesis Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Kinesis Data Streams and Amazon Kinesis Data Firehose problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Kinesis Data Streams and Kinesis Data Firehose when the AWS Console alone isn't enough — shard throughput, consumer lag, producer failures, Firehose delivery issues, Lambda transformations, capacity management, data integrity, encryption, and cross-account access.

### Activate When

- Write or read throughput exceeded (throttling)
- Hot shards from poor partition key distribution
- Resharding (split/merge) failures or stuck operations
- High iterator age (consumer lag)
- KCL checkpoint failures or worker issues
- Enhanced fan-out consumer registration or throughput problems
- Lambda consumer errors or event source mapping issues
- PutRecords partial failures
- KPL aggregation or deaggregation problems
- Firehose S3 delivery failures
- Firehose OpenSearch delivery errors
- Firehose Redshift delivery failures (COPY command errors)
- Firehose HTTP endpoint delivery issues
- Firehose Lambda transformation errors
- Firehose data format conversion problems
- On-demand capacity mode scaling issues
- Provisioned shard count management
- Data loss or missing records investigation
- Record ordering guarantee questions
- Encryption (SSE) configuration issues
- Cross-account stream access problems
- The user says something is wrong with Kinesis without naming specific symptoms

---

## Skill Structure

```
kinesis-troubleshooting/
├── SKILL.md                                    # Main skill definition and investigation workflow
├── README.md                                   # This file
└── references/
    ├── kinesis-guardrails.md                   # 12 anti-misdiagnosis guardrails
    ├── kinesis-hallucination-patterns.yaml     # 8 common LLM hallucination patterns
    ├── A1-write-throttling.md
    ├── A2-read-throttling.md
    ├── A3-hot-shards.md
    ├── A4-resharding-issues.md
    ├── B1-iterator-age-lag.md
    ├── B2-kcl-issues.md
    ├── B3-enhanced-fan-out.md
    ├── B4-lambda-consumer.md
    ├── C1-putrecords-failures.md
    ├── C2-kpl-issues.md
    ├── C3-aggregation-problems.md
    ├── D1-s3-delivery-failures.md
    ├── D2-opensearch-delivery.md
    ├── D3-redshift-delivery.md
    ├── D4-http-endpoint-delivery.md
    ├── E1-lambda-transformation-errors.md
    ├── E2-data-format-conversion.md
    ├── F1-on-demand-scaling.md
    ├── F2-provisioned-shard-management.md
    ├── G1-data-loss-investigation.md
    ├── G2-ordering-guarantees.md
    ├── H1-encryption.md
    ├── H2-cross-account-access.md
    └── Z1-general-troubleshooting.md
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Data Streams Throughput** | A1–A4 | Write throttling, read throttling, hot shards, resharding issues |
| **B — Consumers** | B1–B4 | Iterator age lag, KCL issues, enhanced fan-out, Lambda consumer |
| **C — Producers** | C1–C3 | PutRecords failures, KPL issues, aggregation problems |
| **D — Firehose Delivery** | D1–D4 | S3 delivery failures, OpenSearch delivery, Redshift delivery, HTTP endpoint |
| **E — Firehose Transform** | E1–E2 | Lambda transformation errors, data format conversion |
| **F — Capacity** | F1–F2 | On-demand scaling, provisioned shard management |
| **G — Data Integrity** | G1–G2 | Data loss investigation, ordering guarantees |
| **H — Security** | H1–H2 | Encryption, cross-account access |
| **Z — Catch-All** | Z1 | General Kinesis troubleshooting |

---

## Guardrails & Anti-Hallucination

The skill includes two guardrail files that prevent common misdiagnosis:

### Kinesis Guardrails (`kinesis-guardrails.md`)
12 rules covering: per-shard throughput limits (1 MB/s write, 2 MB/s read), hot shard detection, iterator age = consumer lag, enhanced fan-out vs shared throughput, KCL DynamoDB checkpoint table, Firehose buffering behavior, Firehose Lambda transform response format, Firehose error logging, data retention 24h-365d, resharding shard lineage, PutRecords partial failures, on-demand vs provisioned capacity mode.

### Hallucination Patterns (`kinesis-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about Kinesis, including:
- Per-shard limits are configurable (they are fixed)
- Firehose delivers in real-time (it buffers)
- GetRecords returns deaggregated KPL records (it does not)
- Enhanced fan-out is automatic (requires consumer registration)
- Kinesis retains data indefinitely (24h default, 365d max)
- Firehose supports exactly-once delivery (at-least-once)
- Lambda consumers auto-scale with shards (parallelization factor matters)
- On-demand mode has unlimited throughput (has scaling limits)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using AWS CLI (`describe-stream-summary`, `list-shards`, CloudWatch metrics), CloudTrail events, and application logs. Classify the failure domain.

### Phase 2 — Remediate
Deep dive into the specific domain using targeted commands, CloudWatch metrics, CloudTrail events, or applica