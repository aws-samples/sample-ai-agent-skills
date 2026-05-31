---
title: "D2 — Query Failures"
description: "Diagnose data source query failures and timeouts in Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "query failed"
  - "query timeout"
  - "query error"
  - "data source error"
  - "no data returned"
owner: devops-agent
objective: "Identify why data source queries fail or timeout in AMG and restore data retrieval"
context: "Query failures in AMG can be caused by IAM permission issues, incorrect query syntax, data source misconfiguration, network connectivity problems, or query timeouts due to large data volumes. The root cause depends on the data source type and the specific error message."
---

## Phase 1 — Triage

MUST:
- Identify the data source type and check its configuration via Grafana API
- Check the specific error message from the query (Panel > Inspect > Query)
- Verify the workspace IAM role has permissions for the target data source service
- Test the data source connection: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/datasources/<ds-id>/health`

SHOULD:
- Check if the query works with a simpler version (reduce time range, simplify query)
- Verify the data source endpoint is reachable from the workspace
- Check CloudWatch metrics for the workspace for error patterns

MAY:
- Test the same query directly against the data source service (e.g., CloudWatch API, AMP API)
- Check if the issue is specific to one data source or affects all data sources

## Phase 2 — Remediate

MUST:
- Fix IAM permissions if the error is access-related
- Correct query syntax based on the data source type
- Verify network connectivity if the error is connection-related

SHOULD:
- Optimize queries that timeout by reducing time range or adding filters
- Set appropriate query timeout values in the data source configuration
- Use query caching where available to reduce load

MAY:
- Create recording rules (for Prometheus) to pre-aggregate frequently queried data
- Split complex dashboards into multiple dashboards to reduce query load

## Common Issues

- symptoms: "Query returns 'Access Denied' or 403 error"
  diagnosis: "Workspace IAM role lacks the required permissions for the data source service."
  resolution: "Add the required service permissions to the workspace IAM role."

- symptoms: "Query times out after 30 seconds"
  diagnosis: "Query is too broad (large time range, no filters) or the data source is slow to respond."
  resolution: "Reduce the time range, add filters, or increase the query timeout in data source settings."

- symptoms: "Query returns 'Bad Request' or syntax error"
  diagnosis: "Query syntax is incorrect for the data source type (PromQL, CloudWatch metric math, etc.)."
  resolution: "Fix the query syntax. Use the Explore view to test and validate queries interactively."

## Safety Ratings

```
safety_ratings:
  - "check data source health via API: GREEN — read-only connectivity test"
  - "test query with simpler version: GREEN — read-only query execution"
  - "fix IAM permissions: YELLOW — changes authorization for the workspace role"
  - "optimize queries (reduce time range): GREEN — changes query parameters only"
```

## Escalation Conditions

- All data source queries failing simultaneously, indicating a workspace-level issue
- Query failures affect production alerting rules
- IAM permission changes require cross-team approval
- Query timeouts persist despite optimization, suggesting a data source service issue

## Data Sensitivity

- Query error messages: MEDIUM — may reveal data source endpoints and configuration
- IAM role permissions: MEDIUM — reveals workspace authorization scope
- Query results: HIGH — contain monitoring data from connected services

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER increase query timeout beyond service limits as a permanent fix
- NEVER grant overly broad IAM permissions to resolve query access issues

## Phase 3 — Rollback

1. If IAM permissions were changed, restore previous policy: `aws iam detach-role-policy --role-name <role> --policy-arn <new-policy>` and reattach original
2. If query syntax was changed in dashboards, restore from dashboard JSON backup
3. If data source configuration was modified, restore original settings via Grafana API
4. Verify all affected dashboards and alerts function correctly after rollback
5. Document the query failure pattern for future reference

## Output Format

```yaml
root_cause: "query_failure — <specific_cause>"
evidence:
  - type: error_message
    content: "<specific query error message>"
  - type: data_source
    content: "<data source type and configuration>"
  - type: iam_role
    content: "<relevant IAM permissions>"
severity: HIGH
mitigation:
  immediate: "Fix query syntax, IAM permissions, or data source configuration"
  long_term: "Implement query best practices and monitoring for query performance"
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
