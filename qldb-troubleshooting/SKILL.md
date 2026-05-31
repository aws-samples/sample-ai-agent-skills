---
name: qldb-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon QLDB problems
  by analyzing ledgers, tables, PartiQL queries, journal export,
  verification, streaming, and performance using structured runbooks.
  Activate when: ledger creation failures, PartiQL errors, journal
  export issues, verification failures, streaming problems, or the
  user says something is wrong with QLDB.
compatibility: >
  Requires AWS CLI or SDK access with QLDB, IAM, S3, Kinesis, and
  CloudWatch permissions as applicable.
---

# Amazon QLDB Diagnostics

## When to use

Any Amazon QLDB investigation — ledgers, tables, PartiQL queries, journal export, verification, streaming, or performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws qldb list-ledgers
aws qldb describe-ledger --name <ledger-name>
aws qldb list-journal-kinesis-streams-for-ledger --ledger-name <name>
aws qldb list-journal-s3-exports-for-ledger --ledger-name <name>
```

### Step 2 — Domain deep dive

```
aws qldb-session send-command --session-token <token> --start-session '{"LedgerName":"<name>"}'
aws qldb get-block --name <ledger-name> --block-address '{"IonText":"<address>"}'
aws qldb get-revision --name <ledger-name> --block-address '{"IonText":"<address>"}' --document-id <doc-id>
```

### Step 3 — Detailed investigation

```
aws qldb get-digest --name <ledger-name>
aws cloudwatch get-metric-statistics --namespace AWS/QLDB --metric-name CommandLatency --dimensions Name=LedgerName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=qldb.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any QLDB issue.

## Anti-hallucination rules

1. Always cite specific ledger names, document IDs, or API responses as evidence.
2. QLDB is immutable. Never suggest deleting or modifying journal history.
3. PartiQL is NOT standard SQL. Never assume full SQL compatibility.
4. QLDB uses Ion data format. Never assume JSON-only support.
5. Verification uses cryptographic hashes. Never skip hash validation.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Ledger | A1-A2 | Creation failures, configuration |
| B — Tables | B1-B2 | Table operations, indexes |
| C — PartiQL | C1-C2 | Query errors, syntax issues |
| D — Journal | D1-D2 | Export failures, S3 output |
| E — Verification | E1-E2 | Digest verification, revision proof |
| F — Streaming | F1-F2 | Kinesis streaming, delivery issues |
| G — Performance | G1-G2 | Latency, throughput |
| Z — Catch-All | Z1 | General troubleshooting |
