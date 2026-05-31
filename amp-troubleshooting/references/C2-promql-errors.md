---
title: "C2 — PromQL Errors"
description: "Diagnose and resolve PromQL syntax and evaluation errors in AMP"
status: active
severity: MEDIUM
triggers:
  - "PromQL error"
  - "query error"
  - "parse error"
  - "bad_data"
  - "execution error"
  - "type mismatch"
owner: devops-agent
objective: "Identify and fix PromQL query errors"
context: "AMP supports Prometheus-compatible PromQL. Query errors can be syntax errors (parse failures), type mismatches (wrong function arguments), or evaluation errors (missing metrics, label conflicts). The query API returns descriptive error messages."
---

## Phase 1 — Triage

MUST:
- Test the failing query via curl to get the exact error message:
  ```
  curl -s --aws-sigv4 "aws:amz:<region>:aps" --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" \
    "https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/query?query=<url-encoded-query>"
  ```
- Check the error type in the response: `errorType` (bad_data, execution, timeout)
- Verify the metric name exists: query `{__name__="metric_name"}` to check
- Validate PromQL syntax locally with promtool: `echo 'your_query' | promtool promql query --host=localhost`

SHOULD:
- Check for common syntax issues: unmatched brackets, missing quotes, wrong function names
- Verify label names and values exist in the data
- Check for type mismatches: using instant vector where range vector is expected
- Verify aggregation operators have correct `by` or `without` clauses

MAY:
- Check Prometheus documentation for function signatures
- Test simplified versions of the query to isolate the error
- Review Grafana query editor for syntax highlighting of errors

## Phase 2 — Remediate

MUST:
- Fix syntax errors based on the error message from the API
- Correct type mismatches: use `[5m]` range selector where range vector is required
- Verify metric and label names match exactly (case-sensitive)

SHOULD:
- Use promtool for local query validation before running against AMP
- Build complex queries incrementally — start simple, add complexity
- Use Grafana's query editor which provides autocomplete and error highlighting

MAY:
- Create documentation of commonly-used queries for the team
- Set up query templates in Grafana for standard patterns

## Common Issues

- symptoms: "parse error: unexpected character '...'"
  diagnosis: "PromQL syntax error — invalid character or structure."
  resolution: "Check for unmatched brackets, missing commas, or invalid operators. Validate with promtool."

- symptoms: "bad_data: 1:1: parse error: no expression found in input"
  diagnosis: "Empty or whitespace-only query string."
  resolution: "Ensure the query parameter is not empty. Check URL encoding."

- symptoms: "execution: found duplicate series for the match group"
  diagnosis: "Binary operation matches multiple series on one side — ambiguous join."
  resolution: "Add on() or ignoring() with group_left/group_right to resolve many-to-one matching."

- symptoms: "bad_data: expected type range vector... got instant vector"
  diagnosis: "Function requires a range vector but received an instant vector."
  resolution: "Add a range selector: rate(metric_name[5m]) instead of rate(metric_name)."

## Safety Ratings

```
safety_ratings:
  - "test query via curl: GREEN — read-only query execution"
  - "validate with promtool: GREEN — local syntax validation"
  - "fix PromQL syntax: GREEN — query correction, no data changes"
```

## Escalation Conditions

- PromQL errors affect production alert rules
- Query errors indicate missing metrics, suggesting collector issues
- Binary operation errors require complex query restructuring
- Multiple teams affected by the same PromQL compatibility issue

## Data Sensitivity

- PromQL query text: LOW — query syntax
- Query error messages: LOW — general error information
- Query results: MEDIUM — contain monitoring data

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER modify alert rule expressions without testing the fix first
- NEVER assume PromQL syntax from one Prometheus version works identically in AMP

## Phase 3 — Rollback

1. If alert rule expressions were modified, restore original expressions from backup
2. If dashboard queries were changed, restore from dashboard JSON backup
3. If recording rules were updated with incorrect expressions, restore previous rule file
4. Verify all affected queries return expected results after rollback
5. Test the corrected query in Explore view before applying to dashboards or alerts

## Output Format

```yaml
root_cause: "PromQL error — <specific_cause>"
evidence:
  - type: query_response
    content: "<exact error message from AMP query API>"
  - type: query_text
    content: "<the failing PromQL query>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific PromQL syntax or type error"
  long_term: "Use promtool for validation, implement query templates"
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
