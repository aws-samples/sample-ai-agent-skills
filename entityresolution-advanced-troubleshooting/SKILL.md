---
name: entityresolution-advanced-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot advanced AWS Entity
  Resolution problems by analyzing matching workflow failures, ID
  mapping workflow errors, schema mapping issues, provider service
  errors, rule-based matching tuning, ML matching accuracy,
  output configuration errors, data quality issues, cross-account
  workflows, privacy configuration, and Glue integration.
  Activate when: matching workflow failures, ID mapping errors,
  schema mapping issues, provider service problems, matching
  accuracy concerns, output configuration errors, data quality
  problems, cross-account workflow failures, privacy configuration
  issues, Glue integration errors, or the user says something is
  wrong with advanced Entity Resolution features.
compatibility: >
  Requires AWS CLI or SDK access with entityresolution, glue, s3,
  iam, cloudtrail, cloudwatch, and kms permissions.
---

# AWS Entity Resolution Advanced Diagnostics

## When to use

Any advanced Entity Resolution investigation — matching workflow failures, ID mapping errors, schema mapping issues, provider service errors, rule-based matching tuning, ML matching accuracy, output configuration, data quality, cross-account workflows, privacy configuration, or Glue integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws entityresolution list-matching-workflows --query 'workflowSummaries[*].{Name:workflowName,Arn:workflowArn,CreatedAt:createdAt}'
aws entityresolution list-id-mapping-workflows --query 'workflowSummaries[*].{Name:workflowName,Arn:workflowArn}'
aws entityresolution list-schema-mappings --query 'schemaList[*].{Name:schemaName,Arn:schemaArn}'
```

### Step 2 — Domain deep dive

```
aws entityresolution get-matching-workflow --workflow-name <workflow-name>
aws entityresolution get-matching-job --workflow-name <workflow-name> --job-id <job-id>
aws entityresolution get-schema-mapping --schema-name <schema-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=entityresolution.amazonaws.com --max-results 20
aws entityresolution get-id-mapping-workflow --workflow-name <workflow-name>
aws glue get-table --database-name <db> --name <table>
```

Read `references/guardrails.md` before concluding on any advanced Entity Resolution issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `entityresolution list-matching-workflows` | List matching workflows |
| `entityresolution get-matching-workflow` | Get workflow configuration |
| `entityresolution get-matching-job` | Get job execution details |
| `entityresolution list-id-mapping-workflows` | List ID mapping workflows |
| `entityresolution get-schema-mapping` | Get schema mapping details |
| `entityresolution list-provider-services` | List available provider services |
| `glue get-table` | Check Glue table schema for input data |

## Gotchas: Entity Resolution Advanced

- Matching workflows support THREE resolution types: RULE_MATCHING, ML_MATCHING, and PROVIDER. Each has different configuration requirements, accuracy characteristics, and pricing.
- ID mapping workflows are separate from matching workflows. They map identifiers across datasets using provider services, not the same matching logic as matching workflows.
- Schema mappings define how input data fields map to Entity Resolution attributes. Incorrect mappings cause silent matching failures — records are processed but not matched correctly.
- Provider services (LiveRamp, TransUnion, etc.) require separate subscription and configuration. Provider errors may be due to subscription issues, not Entity Resolution configuration.
- Rule-based matching uses ordered rules evaluated sequentially. Rule order matters — more specific rules should come first. Overly broad rules cause false positives.
- ML matching accuracy depends on data quality and volume. Small datasets or datasets with many missing fields produce poor results. ML matching requires a minimum data threshold.
- Output configuration determines where results are written. S3 output requires proper bucket permissions. Glue output requires Glue database and table permissions.

## Anti-hallucination rules

1. Always cite specific workflow names, job IDs, or API responses as evidence.
2. RULE_MATCHING, ML_MATCHING, and PROVIDER are different resolution types with different behaviors. Never conflate them.
3. Matching workflows and ID mapping workflows are separate features. Never mix them up.
4. Provider services have their own error modes independent of Entity Resolution. Never blame Entity Resolution for provider issues.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Workflows | A1-A2 | Matching workflow failures, ID mapping workflow errors |
| B — Configuration | B1-B2 | Schema mapping issues, provider service errors |
| C — Matching | C1-C2 | Rule-based matching tuning, ML matching accuracy |
| D — Data | D1-D2 | Output configuration errors, data quality issues |
| E — Advanced | E1-E2 | Cross-account workflows, privacy configuration |
| F — Integration | F1 | Glue integration |
| Z — Catch-All | Z1 | General troubleshooting |
