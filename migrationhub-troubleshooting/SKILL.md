---
name: migrationhub-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Migration Hub problems
  by analyzing discovery, assessment, migration tracking, strategy
  recommendations, refactor spaces, and application grouping using
  structured runbooks.
  Activate when: discovery failures, assessment errors, migration tracking
  issues, strategy recommendation problems, refactor spaces failures,
  application grouping errors, or the user says something is wrong with
  Migration Hub without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Migration Hub, Migration Hub Strategy,
  Migration Hub Refactor Spaces, Application Discovery Service, CloudWatch,
  CloudTrail permissions.
---

# Migration Hub Diagnostics

## When to use

Any AWS Migration Hub investigation — discovery, assessment, migration tracking, strategy recommendations, refactor spaces, or application grouping.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mgh describe-home-region
aws mgh list-migration-tasks --max-results 20
aws mgh list-discovered-resources --migration-task-name <task> --max-results 20
aws migrationhub-strategy get-portfolio-summary
```

### Step 2 — Domain deep dive

```
aws mgh describe-migration-task --migration-task-name <task> --progress-update-stream <stream>
aws mgh list-progress-update-streams --max-results 20
aws migrationhub-strategy list-servers --max-results 20
aws migrationhub-strategy get-server-details --server-id <id>
```

### Step 3 — Detailed investigation

```
aws migration-hub-refactor-spaces list-environments --max-results 20
aws migration-hub-refactor-spaces get-environment --environment-identifier <id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mgh.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any Migration Hub issue.

## Gotchas: Migration Hub

- Migration Hub has a home region that must be set before use. All data is stored in the home region.
- Migration tracking aggregates status from multiple migration tools (DMS, SMS, CloudEndure, etc.).
- Strategy Recommendations requires data collection agents or manual import.
- Refactor Spaces manages the infrastructure for incremental refactoring (strangler fig pattern).
- Application grouping is logical — it doesn't affect the actual migration.
- Progress update streams are per-migration-tool, not per-application.

## Anti-hallucination rules

1. Home region must be set before any Migration Hub operations.
2. Migration Hub doesn't perform migrations — it tracks status from other tools.
3. Strategy Recommendations requires collected server data before generating recommendations.
4. Refactor Spaces creates real AWS resources (API Gateway, NLB, Lambda, Transit Gateway).
5. Application Discovery Service is separate from Migration Hub but feeds data to it.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Discovery | A1-A2 | Data collection, import |
| B — Assessment | B1-B2 | Portfolio assessment, strategy |
| C — Tracking | C1-C3 | Migration tasks, progress, status |
| D — Strategy | D1-D2 | Recommendations, analysis |
| E — Refactor | E1-E2 | Refactor spaces, routing |
| F — Grouping | F1 | Application grouping |
| Z — Catch-All | Z1 | General troubleshooting |
