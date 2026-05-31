---
title: "C1 — Query Failures"
description: "Diagnose Timestream query execution failures"
status: active
severity: HIGH
triggers:
  - "query failed"
  - "query error"
  - "SQL syntax error"
  - "cannot query Timestream"
owner: devops-agent
objective: "Identify and resolve query execution failures"
context: "Timestream uses SQL for queries. Failures can be caused by syntax errors, permission issues, non-existent tables, query timeout, or exceeding query limits. Table names must be double-quoted in queries."
---

## Phase 1 — Triage

MUST:
- Verify the query syntax (Timestream SQL dialect)
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names timestream:Select timestream:DescribeEndpoints`
- Verify the table exists: `aws timestream-write describe-table --database-name <database> --table-name <table>`
- Test a simple query: `aws timestream-query query --query-string "SELECT count(*) FROM \"<database>\".\"<table>\" WHERE time > ago(1h)"`

SHOULD:
- Check for query timeout (default 60 seconds for interactive queries)
- Verify the query endpoint is accessible: `aws timestream-query describe-endpoints`
- Check CloudWatch for query errors

MAY:
- Review query complexity and optimize
- Check account-level query concurrency limits

## Phase 2 — Remediate

MUST:
- Fix SQL syntax errors (use double quotes for table/database names)
- Fix IAM permissions for timestream:Select
- Ensure the query endpoint is accessible

SHOULD:
- Optimize queries to avoid timeouts (use time filters, limit results)
- Use parameterized queries for repeated patterns

MAY:
- Request query concurrency quota increase
- Create scheduled queries for complex aggregations

## Common Issues

- symptoms: "ValidationException — syntax error"
  diagnosis: "SQL syntax error or unquoted table/database names."
  resolution: "Use double quotes: SELECT * FROM \"database\".\"table\" WHERE time > ago(1h)."

- symptoms: "AccessDeniedException on query"
  diagnosis: "IAM role missing timestream:Select permission."
  resolution: "Add timestream:Select and timestream:DescribeEndpoints to the role."

- symptoms: "Query timeout"
  diagnosis: "Query too complex or scanning too much data."
  resolution: "Add time filters, reduce result set, or use scheduled queries for aggregations."

## Output Format

```yaml
root_cause: "query_failure — <specific_cause>"
evidence:
  - type: query_error
    content: "<error message>"
  - type: iam_permissions
    content: "<IAM evaluation>"
  - type: table_status
    content: "<table existence and status>"
severity: HIGH
mitigation:
  immediate: "Fix query syntax or permissions"
  long_term: "Optimize queries and use scheduled queries for heavy aggregations"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
