---
name: gluedatabrew-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Glue DataBrew problems
  by analyzing projects, recipe jobs, profile jobs, datasets, data quality
  rules, schedules, output formats, S3 permissions, and following structured
  runbooks. Activate when: project creation failures, recipe job errors,
  profile job failures, dataset connection issues, data quality rule failures,
  schedule problems, output format errors, S3 permission denied, or the user
  says something is wrong with DataBrew without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with databrew:*, S3, IAM, CloudWatch,
  and CloudTrail permissions.
---

# AWS Glue DataBrew Diagnostics

## When to use

Any AWS Glue DataBrew investigation — project setup, recipe authoring, profile jobs, dataset configuration, data quality, scheduling, output formatting, or S3 access issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws databrew list-projects
aws databrew list-jobs
aws databrew list-datasets
aws databrew list-schedules
aws databrew list-rulesets
```

### Step 2 — Domain deep dive

```
aws databrew describe-project --name <project-name>
aws databrew describe-job --name <job-name>
aws databrew describe-dataset --name <dataset-name>
aws databrew list-job-runs --name <job-name> --max-results 5
```

### Step 3 — Detailed investigation

```
aws databrew describe-job-run --name <job-name> --run-id <run-id>
aws logs filter-log-events --log-group-name /aws-glue-databrew/<job-name> --limit 50
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=databrew.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any DataBrew issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-projects` | Enumerate projects |
| `describe-job` | Check job configuration |
| `list-job-runs` | Check recent job executions |
| `describe-job-run` | Get job run details and errors |
| `describe-dataset` | Check dataset configuration |
| `list-rulesets` | Check data quality rules |
| CloudWatch Logs | Analyze job execution logs |
| CloudTrail | Audit configuration changes |

## Gotchas: AWS Glue DataBrew

- DataBrew is a visual data preparation tool, NOT the same as AWS Glue ETL. They share the Glue service but have different APIs, IAM actions, and job types.
- DataBrew jobs require an IAM role with S3 access. The role must have both read access to input and write access to output locations.
- Recipe jobs apply transformations; profile jobs analyze data. They are different job types with different configurations and outputs.
- DataBrew datasets reference S3 locations, database connections, or Data Catalog tables. The dataset itself doesn't store data.
- Data quality rulesets are separate from recipes. They define validation rules checked during profile jobs.
- Output format and compression are configured per job, not per project. Each job can have different output settings.

## Anti-hallucination rules

1. Always cite specific project names, job names, or API responses as evidence.
2. DataBrew APIs use `databrew:` prefix, NOT `glue:`. Never confuse DataBrew IAM actions with Glue ETL actions.
3. Recipe steps are NOT Glue ETL transforms. Never suggest Glue ETL syntax for DataBrew recipes.
4. DataBrew does NOT support streaming data. It processes batch data only.
5. Profile jobs and recipe jobs are different. Never confuse their configurations.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## Runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Projects | A1–A2 | Project creation, project configuration |
| B — Recipe Jobs | B1–B2 | Recipe job failures, recipe steps |
| C — Profile Jobs | C1–C2 | Profile job failures, data quality |
| D — Datasets | D1–D2 | Dataset creation, connection issues |
| E — Output | E1–E2 | Output format, S3 permissions |
| F — Scheduling | F1–F2 | Schedule creation, schedule failures |
| Z — Catch-All | Z1 | General troubleshooting |
