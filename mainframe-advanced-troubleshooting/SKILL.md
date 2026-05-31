---
name: mainframe-advanced-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Mainframe
  Modernization advanced problems by analyzing Blu Age refactoring,
  Micro Focus replatforming, application deployment, runtime environments,
  batch jobs, CICS transactions, data migration, dataset management,
  performance tuning, monitoring, and hybrid connectivity using
  structured runbooks.
  Activate when: Blu Age refactoring errors, Micro Focus replatforming
  issues, deployment failures, runtime environment problems, batch job
  failures, CICS transaction errors, data migration issues, performance
  problems, or the user says something is wrong with Mainframe Modernization.
compatibility: >
  Requires AWS CLI or SDK access with m2, s3, cloudwatch, logs,
  ec2, and iam permissions. Mainframe domain knowledge (COBOL, JCL,
  CICS, VSAM) recommended for effective troubleshooting.
---

# AWS Mainframe Modernization Advanced Diagnostics

## When to use

Any AWS Mainframe Modernization advanced investigation — Blu Age refactoring, Micro Focus replatforming, application deployment, runtime environments, batch jobs, CICS transactions, data migration, dataset management, performance tuning, monitoring integration, or hybrid connectivity.

## Investigation workflow

### Step 1 — Collect and triage

```
aws m2 list-environments --query 'environments[*].{Name:name,Id:environmentId,EngineType:engineType,Status:status}'
aws m2 list-applications --query 'applications[*].{Name:name,Id:applicationId,EngineType:engineType,Status:status}'
aws m2 list-deployments --application-id <app-id> --query 'deployments[*].{Id:deploymentId,Status:status,CreationTime:creationTime}'
```

### Step 2 — Domain deep dive

```
aws m2 get-environment --environment-id <env-id>
aws m2 get-application --application-id <app-id>
aws m2 get-deployment --application-id <app-id> --deployment-id <deployment-id>
```

### Step 3 — Detailed investigation

```
aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --start-time <epoch-ms> --filter-pattern "ERROR"
aws m2 list-batch-job-executions --application-id <app-id> --query 'batchJobExecutions[*].{Id:executionId,Status:status,JobName:jobName}'
aws cloudwatch get-metric-statistics --namespace AWS/M2 --metric-name CPUUtilization --dimensions Name=EnvironmentId,Value=<env-id> --start-time <start> --end-time <end> --period 300 --statistics Average
```

Read `references/guardrails.md` before concluding on any Mainframe Modernization issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `m2 list-environments` | List runtime environments |
| `m2 get-environment` | Get environment details |
| `m2 list-applications` | List applications |
| `m2 get-application` | Get application details |
| `m2 list-deployments` | List deployment history |
| `m2 list-batch-job-executions` | List batch job runs |
| `m2 get-batch-job-execution` | Get batch job details |

## Gotchas: AWS Mainframe Modernization

- Blu Age (refactoring) and Micro Focus (replatforming) are DIFFERENT engines with different capabilities. Blu Age converts COBOL to Java. Micro Focus runs COBOL natively. Do not mix their troubleshooting approaches.
- Runtime environments are the compute infrastructure. Applications are deployed TO environments. An environment must be in Available status before deploying applications.
- Batch jobs in Mainframe Modernization map to JCL jobs. They have different execution semantics than Lambda or Step Functions. Job dependencies and scheduling follow mainframe patterns.
- CICS transactions in Micro Focus environments require CICS region configuration. Transaction routing, program definitions, and resource definitions must be correctly configured.
- Data migration from mainframe datasets (VSAM, sequential, GDG) to AWS requires careful mapping. EBCDIC to ASCII conversion, record format handling, and copybook parsing are common failure points.
- Mainframe Modernization applications use S3 for artifact storage. Application definitions, deployment packages, and data files are stored in S3 buckets with specific structure requirements.
- Hybrid connectivity between on-premises mainframes and AWS M2 environments requires VPN or Direct Connect. Data synchronization during migration phases needs careful orchestration.

## Anti-hallucination rules

1. Always cite specific environment IDs, application IDs, or deployment IDs as evidence.
2. Blu Age (refactoring to Java) and Micro Focus (replatforming COBOL) are DIFFERENT engines. Never conflate them.
3. Mainframe concepts (JCL, CICS, VSAM) have specific meanings. Never approximate with generic cloud equivalents.
4. Batch job execution follows mainframe semantics. Never assume serverless or container execution patterns.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Modernization Engines | A1–A2 | Blu Age refactoring errors, Micro Focus replatforming |
| B — Deployment & Runtime | B1–B2 | Application deployment failures, runtime environment issues |
| C — Workloads | C1–C2 | Batch job failures, CICS transaction errors |
| D — Data | D1–D2 | Data migration issues, dataset management |
| E — Observability | E1–E2 | Performance tuning, monitoring integration |
| F — Connectivity | F1 | Hybrid connectivity |
| Z — Catch-All | Z1 | General troubleshooting |
