---
name: mainframe-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Mainframe Modernization
  problems by analyzing application creation, runtime environments, batch jobs,
  data sets, CICS/IMS transactions, refactoring, and replatforming using
  structured runbooks.
  Activate when: application creation failures, runtime environment issues,
  batch job failures, data set problems, CICS/IMS errors, refactoring issues,
  replatforming problems, or the user says something is wrong with Mainframe
  Modernization without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with M2 (Mainframe Modernization),
  S3, CloudWatch, CloudTrail, EC2, EFS permissions.
---

# Mainframe Modernization Diagnostics

## When to use

Any AWS Mainframe Modernization investigation — application creation, runtime environments, batch jobs, data sets, CICS/IMS, refactoring, or replatforming.

## Investigation workflow

### Step 1 — Collect and triage

```
aws m2 list-environments --max-results 20
aws m2 list-applications --max-results 20
aws m2 get-environment --environment-id <env-id>
aws m2 get-application --application-id <app-id>
```

### Step 2 — Domain deep dive

```
aws m2 list-batch-job-executions --application-id <app-id> --max-results 10
aws m2 get-batch-job-execution --application-id <app-id> --execution-id <exec-id>
aws m2 list-data-sets --application-id <app-id> --max-results 20
aws m2 list-deployments --application-id <app-id> --max-results 10
```

### Step 3 — Detailed investigation

```
aws m2 get-deployment --application-id <app-id> --deployment-id <deploy-id>
aws m2 get-data-set-details --application-id <app-id> --data-set-name <name>
aws cloudwatch get-metric-statistics --namespace AWS/M2 --metric-name CPUUtilization --dimensions Name=EnvironmentId,Value=<env-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=m2.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any Mainframe Modernization issue.

## Gotchas: Mainframe Modernization

- Two engines: Micro Focus (replatforming) and Blu Age (refactoring/automated).
- Runtime environments are managed compute for running mainframe applications.
- Batch jobs run COBOL/JCL programs in the managed environment.
- Data sets are mainframe-style data storage managed by the service.
- CICS and IMS transaction processing is supported in Micro Focus engine.
- Applications must be deployed to an environment before they can run.

## Anti-hallucination rules

1. Micro Focus and Blu Age engines have different capabilities and use cases.
2. Runtime environments must be created before deploying applications.
3. Data sets must be imported or created before batch jobs can reference them.
4. CICS/IMS support is Micro Focus engine only, not Blu Age.
5. Application deployment is a separate step from application creation.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Application | A1-A2 | App creation, deployment |
| B — Runtime | B1-B2 | Environment creation, config |
| C — Batch | C1-C2 | Batch jobs, JCL execution |
| D — Data | D1-D2 | Data sets, import/export |
| E — CICS/IMS | E1-E2 | Transaction processing, config |
| F — Refactoring | F1 | Blu Age refactoring |
| G — Replatforming | G1-G2 | Micro Focus replatforming, testing |
| Z — Catch-All | Z1 | General troubleshooting |
