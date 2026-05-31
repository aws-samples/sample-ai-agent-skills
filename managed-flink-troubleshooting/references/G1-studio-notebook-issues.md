---
title: "G1 — Studio Notebook Issues"
description: "Diagnose Managed Flink Studio notebook (Zeppelin) issues"
status: active
severity: MEDIUM
triggers: ["Studio notebook error", "Zeppelin error", "notebook not starting", "notebook query failed"]
owner: devops-agent
objective: "Resolve Studio notebook issues including startup, query execution, resource limits, and deployment"
context: "Studio notebooks use Apache Zeppelin with Flink interpreter. Designed for interactive development. Different resource limits than code-based applications."
---

## Phase 1 — Triage

MUST:
- Check notebook application status: `aws kinesisanalyticsv2 describe-application --application-name <notebook-name>`
- Verify runtime is ZEPPELIN-FLINK: check RuntimeEnvironment in application details
- Review CloudWatch Logs: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<notebook-name> --start-time <epoch-ms> --limit 50`

SHOULD:
- Check notebook resource allocation (KPUs)
- Verify IAM role has permissions for data sources

MAY:
- Check Zeppelin interpreter configuration
- Review notebook paragraph execution history

## Phase 2 — Remediate

MUST:
- Ensure notebook application is in RUNNING state before executing queries
- Fix IAM role permissions for source/sink access
- Verify Flink SQL syntax is correct for the Flink version

SHOULD:
- Increase KPUs if queries fail with resource exhaustion
- Use appropriate catalog configuration for data sources

MAY:
- Export working queries to code-based application for production
- Configure custom Flink properties via Zeppelin interpreter settings

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "logs filter-log-events: GREEN — read-only log review"
  - "StartApplication (notebook): YELLOW — starts notebook and allocates KPUs"
  - "StopApplication (notebook): YELLOW — stops notebook and releases KPUs"
```

## Escalation Conditions

- Notebook fails to start with internal service errors
- Query execution causes notebook to become unresponsive
- Notebook requires more resources than Studio limits allow
- Production deployment requires features not available in Studio

## Data Sensitivity

- `aws kinesisanalyticsv2 describe-application`: MEDIUM — reveals notebook configuration
- `aws logs filter-log-events`: HIGH — may contain query results and data samples
- Notebook paragraphs: HIGH — contain SQL queries and potentially sensitive data

## Prohibited Actions

- NEVER use Studio notebooks for production streaming workloads
- NEVER store sensitive data in notebook paragraphs
- NEVER leave notebooks running when not in use (KPU charges continue)
- NEVER assume Studio notebook features match code-based application capabilities

## Phase 3 — Rollback

1. If notebook configuration was changed, revert to previous settings
2. If notebook is unresponsive, stop and restart it
3. Verify data sources are accessible after notebook restart
4. Export important queries before making destructive changes
5. Document notebook configuration and query patterns

## Common Issues

- symptoms: "Flink SQL query returns 'Table not found' in Studio notebook"
  diagnosis: "Catalog or connector configuration is missing. Tables must be defined in the notebook session."
  resolution: "Create table definitions using CREATE TABLE DDL with connector properties. Verify source stream/topic exists."

## Output Format

```yaml
root_cause: "studio_notebook — <specific_cause>"
evidence:
  - type: notebook_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix notebook issue"
  long_term: "Export to code-based app"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
