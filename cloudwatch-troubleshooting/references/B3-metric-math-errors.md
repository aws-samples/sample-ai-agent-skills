---
title: "B3 — Metric Math Errors"
description: "Diagnose metric math expression failures and unexpected results"
status: active
severity: MEDIUM
triggers:
  - "metric math"
  - "METRICS function"
  - "math expression"
  - "metric calculation"
  - "expression error"
owner: devops-agent
objective: "Identify why a metric math expression is returning errors or unexpected results"
context: "Metric math allows combining metrics with mathematical expressions. Common issues include syntax errors, missing data causing entire expression to return missing, mismatched periods, referencing non-existent metrics, and SEARCH expression limitations. The FILL function can handle missing data."
---

## Phase 1 — Triage

MUST:
- Check the metric math expression syntax in the alarm or dashboard
- Verify all referenced metrics exist and have data: `aws cloudwatch get-metric-data --metric-data-queries file://queries.json --start-time <iso> --end-time <iso>`
- Check if any input metric has missing data for the evaluation period
- Verify all metrics use the same period

SHOULD:
- Test the expression with get-metric-data to see actual results
- Check if SEARCH expressions are returning expected metrics
- Verify metric IDs in the expression match the query definitions

MAY:
- Simplify the expression to isolate which part is failing
- Check for division by zero scenarios

## Phase 2 — Remediate

MUST:
- Fix expression syntax errors
- Use FILL() function to handle missing data: `FILL(m1, 0)` or `FILL(m1, REPEAT)`
- Ensure all referenced metrics exist and publish data

SHOULD:
- Align periods across all input metrics
- Use IF() function for conditional logic instead of complex expressions
- Test expressions in the CloudWatch console metric math editor

MAY:
- Document complex expressions for operational clarity
- Set up separate alarms for individual metrics as fallback

## Common Issues

- symptoms: "Metric math alarm stuck in INSUFFICIENT_DATA"
  diagnosis: "One or more input metrics have missing data, causing the expression result to be missing."
  resolution: "Use FILL(metric, value) to provide default values for missing data."

- symptoms: "Expression returns unexpected values"
  diagnosis: "Metrics have different periods causing misalignment."
  resolution: "Ensure all metrics in the expression use the same period."

- symptoms: "SEARCH expression returns no results"
  diagnosis: "SEARCH pattern does not match any existing metrics."
  resolution: "Verify the namespace, metric name, and dimension patterns in the SEARCH expression."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix expression syntax | GREEN | Configuration fix, no state change |
| Add FILL() for missing data | GREEN | Improves expression resilience |
| Align metric periods | GREEN | Configuration fix |

## Escalation Conditions

- Metric math alarm stuck in INSUFFICIENT_DATA for production monitoring — escalate to monitoring team
- Complex metric math expressions producing incorrect values — escalate to dashboard/alarm owner
- SEARCH expressions not matching expected metrics — escalate to metric publishing team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch describe-alarms` | MEDIUM | Exposes alarm configuration and thresholds |
| `aws cloudwatch get-metric-data` | LOW | Numeric metric data |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** modify metric math expressions on production alarms without testing first

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed expression syntax | Restore previous expression |
| Added FILL() function | Remove FILL() if causing incorrect behavior |
| Aligned metric periods | Restore previous period settings |

## Output Format

```yaml
root_cause: "metric_math — <specific_cause>"
evidence:
  - type: expression
    content: "<metric math expression>"
  - type: metric_data
    content: "<get-metric-data results>"
severity: MEDIUM
mitigation:
  immediate: "Fix expression syntax, add FILL for missing data, or align periods"
  long_term: "Document metric math expressions and test with get-metric-data"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
