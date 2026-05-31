---
name: finspace-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon FinSpace problems
  by analyzing environment configuration, datasets, managed kdb, permissions,
  superuser access, and changeset management using structured runbooks.
  Activate when: environment creation failures, dataset issues, managed kdb
  problems, permission errors, superuser access issues, changeset failures,
  or the user says something is wrong with FinSpace without naming specific
  symptoms.
compatibility: >
  Requires AWS CLI or SDK access with FinSpace, S3, IAM, CloudWatch,
  CloudTrail permissions.
---

# FinSpace Diagnostics

## When to use

Any Amazon FinSpace investigation — environment configuration, datasets, managed kdb, permissions, superuser access, or changeset management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws finspace list-environments --max-results 20
aws finspace get-environment --environment-id <env-id>
aws finspace list-kx-environments --max-results 20
```

### Step 2 — Domain deep dive

```
aws finspace get-kx-environment --environment-id <env-id>
aws finspace list-kx-databases --environment-id <env-id> --max-results 20
aws finspace list-kx-clusters --environment-id <env-id> --max-results 20
aws finspace list-kx-changesets --environment-id <env-id> --database-name <db> --max-results 10
```

### Step 3 — Detailed investigation

```
aws finspace get-kx-cluster --environment-id <env-id> --cluster-name <name>
aws finspace get-kx-changeset --environment-id <env-id> --database-name <db> --changeset-id <id>
aws finspace get-kx-database --environment-id <env-id> --database-name <db>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=finspace.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any FinSpace issue.

## Gotchas: FinSpace

- FinSpace has two modes: managed datasets (legacy) and managed kdb Insights.
- Managed kdb provides kdb+ database clusters for time-series analytics.
- Environments are isolated workspaces with their own networking and permissions.
- Changesets are immutable data updates to kdb databases.
- Superuser has full access to the environment — use sparingly.
- kdb clusters have specific node types optimized for different workloads.

## Anti-hallucination rules

1. FinSpace managed datasets and managed kdb are different features.
2. kdb environments require VPC configuration with specific networking.
3. Changesets are immutable — you can't modify a changeset after creation.
4. Superuser credentials should be rotated and access minimized.
5. kdb cluster types (HDB, RDB, Gateway, GP) serve different purposes.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Environment | A1-A2 | Environment creation, config |
| B — Datasets | B1-B2 | Dataset management, views |
| C — Kdb | C1-C2 | Kdb clusters, databases |
| D — Permissions | D1-D2 | Access control, IAM |
| E — Superuser | E1 | Superuser management |
| F — Changesets | F1 | Changeset management |
| Z — Catch-All | Z1 | General troubleshooting |
