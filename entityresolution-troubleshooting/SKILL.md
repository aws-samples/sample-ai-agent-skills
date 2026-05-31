---
name: entityresolution-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Entity Resolution
  problems by analyzing schema mappings, matching workflows, ID mapping
  workflows, provider services, output configuration, rule-based matching,
  ML matching, and following structured runbooks. Activate when: schema
  mapping errors, matching workflow failures, ID mapping issues, provider
  service configuration, output problems, or the user says something is
  wrong with Entity Resolution without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with entityresolution:*, S3, IAM, Glue
  Data Catalog, and CloudTrail permissions.
---

# AWS Entity Resolution Diagnostics

## When to use

Any AWS Entity Resolution investigation — schema mapping, matching workflows, ID mapping, provider services, output configuration, or matching algorithm issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws entityresolution list-schema-mappings
aws entityresolution list-matching-workflows
aws entityresolution list-id-mapping-workflows
aws entityresolution list-provider-services
```

### Step 2 — Domain deep dive

```
aws entityresolution get-schema-mapping --schema-name <name>
aws entityresolution get-matching-workflow --workflow-name <name>
aws entityresolution get-id-mapping-workflow --workflow-name <name>
aws entityresolution list-matching-jobs --workflow-name <name>
```

### Step 3 — Detailed investigation

```
aws entityresolution get-matching-job --workflow-name <name> --job-id <id>
aws entityresolution get-id-mapping-job --workflow-name <name> --job-id <id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=entityresolution.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Entity Resolution issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-schema-mappings` | Enumerate schema mappings |
| `get-matching-workflow` | Check matching workflow config |
| `list-matching-jobs` | Check job execution history |
| `get-matching-job` | Get job details and errors |
| `get-id-mapping-workflow` | Check ID mapping config |
| CloudTrail | Audit configuration changes |

## Gotchas: AWS Entity Resolution

- Schema mappings define how input data columns map to Entity Resolution attributes. They reference Glue Data Catalog tables.
- Matching workflows use either rule-based or ML-based matching. Rule-based requires explicit match rules; ML-based uses AWS-trained models.
- ID mapping workflows map identities across datasets using provider services (e.g., LiveRamp, TransUnion).
- Provider services are third-party integrations that require separate subscriptions and configuration.
- Output goes to S3. The IAM role must have write access to the output location.
- Entity Resolution does NOT modify source data. It produces match results as output files.

## Anti-hallucination rules

1. Always cite specific workflow names, job IDs, or API responses as evidence.
2. Entity Resolution is NOT Glue ETL. They are separate services.
3. Schema mappings reference Glue Data Catalog, NOT raw S3 files directly.
4. ML matching is NOT customizable — it uses AWS pre-trained models.
5. Provider services require third-party subscriptions.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## Runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Schema | A1–A2 | Schema mapping creation, mapping validation |
| B — Matching | B1–B2 | Matching workflow, rule-based matching |
| C — ML Matching | C1–C2 | ML matching config, match quality |
| D — ID Mapping | D1–D2 | ID mapping workflow, provider services |
| E — Output | E1–E2 | Output configuration, output analysis |
| F — Jobs | F1–F2 | Job execution, job failures |
| Z — Catch-All | Z1 | General troubleshooting |
