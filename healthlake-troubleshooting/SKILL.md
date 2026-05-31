---
name: healthlake-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS HealthLake problems
  by analyzing data stores, FHIR import/export, search operations,
  analytics, NLP enrichment, and security configuration using structured
  runbooks. Activate when: data store creation failures, FHIR import
  errors, export failures, search issues, analytics problems, NLP
  enrichment errors, or the user says something is wrong with HealthLake.
compatibility: >
  Requires AWS CLI or SDK access with HealthLake, IAM, S3, CloudWatch,
  and KMS permissions as applicable.
---

# AWS HealthLake Diagnostics

## When to use

Any AWS HealthLake investigation — data stores, FHIR import/export, search, analytics, NLP enrichment, or security troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws healthlake list-fhir-datastores
aws healthlake describe-fhir-datastore --datastore-id <id>
```

### Step 2 — Domain deep dive

```
aws healthlake list-fhir-import-jobs --datastore-id <id>
aws healthlake describe-fhir-import-job --datastore-id <id> --job-id <job-id>
aws healthlake list-fhir-export-jobs --datastore-id <id>
aws healthlake describe-fhir-export-job --datastore-id <id> --job-id <job-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=healthlake.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/HealthLake --metric-name SuccessfulRequests --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any HealthLake issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-fhir-datastore` | Check data store status |
| `describe-fhir-import-job` | Check import job status |
| `describe-fhir-export-job` | Check export job status |
| FHIR REST API | Test CRUD and search operations |
| CloudWatch Metrics | Check request rates and errors |

## Anti-hallucination rules

1. Always cite specific data store IDs, job IDs, or API responses as evidence.
2. HealthLake uses FHIR R4 standard. Never assume other FHIR versions.
3. Import/export jobs are asynchronous. Never assume instant completion.
4. NLP enrichment is automatic when enabled. Never suggest manual NLP.
5. Data stores use KMS encryption. Never assume unencrypted storage.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Data Store | A1-A2 | Creation failures, configuration |
| B — Import | B1-B2 | FHIR import errors, data validation |
| C — Export | C1-C2 | Export failures, S3 output |
| D — Search | D1-D2 | Search queries, performance |
| E — Analytics | E1-E2 | Analytics queries, Lake Formation |
| F — NLP | F1-F2 | NLP enrichment, entity extraction |
| G — Security | G1-G2 | Encryption, access control |
| Z — Catch-All | Z1 | General troubleshooting |
