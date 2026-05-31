---
title: "E2 — CloudWatch Data Source"
description: "Diagnose and resolve CloudWatch data source issues for Lookout for Metrics"
status: active
severity: MEDIUM
triggers: ["CloudWatch data source error", "no CloudWatch metrics", "CloudWatch connectivity failed", "metric not found"]
owner: devops-agent
objective: "Ensure reliable data ingestion from CloudWatch metrics into Lookout for Metrics detectors"
context: "CloudWatch data sources require exact metric namespace, name, and dimension matches. Partial matches, wildcards, and regex are not supported. Metric existence must be verified before configuration."
---

## Phase 1 — Triage

MUST:
- Describe metric set to get CloudWatch configuration: `aws lookoutmetrics describe-metric-set --metric-set-arn <arn>`
- Verify CloudWatch metrics exist with exact names: `aws cloudwatch list-metrics --namespace <namespace> --metric-name <name>`
- Check metric dimensions match exactly: `aws cloudwatch list-metrics --namespace <namespace> --metric-name <name> --dimensions Name=<dim_name>,Value=<dim_value>`

SHOULD:
- Verify data points exist in the expected time range: `aws cloudwatch get-metric-statistics --namespace <namespace> --metric-name <name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check data quality metrics: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`

MAY:
- Review CloudWatch metric publishing frequency vs detector frequency
- Check for CloudWatch metric math or custom metrics that may have different behavior

## Phase 2 — Remediate

MUST:
- Correct metric namespace, name, and dimensions to match exactly
- Ensure IAM role has `cloudwatch:GetMetricData` and `cloudwatch:ListMetrics` permissions
- Verify the metric has data points at the detector's configured frequency

SHOULD:
- Align detector frequency with CloudWatch metric publishing frequency
- Verify cross-account CloudWatch access if metrics are in a different account
- Check for CloudWatch metric retention period (data older than 15 months is not available)

MAY:
- Create CloudWatch metric filters if metrics need to be derived from logs
- Set up CloudWatch metric alarms as a secondary monitoring mechanism

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
  - "cloudwatch get-metric-statistics: GREEN — read-only data retrieval"
  - "describe-metric-set: GREEN — read-only configuration check"
  - "IAM role policy updates: YELLOW — modifies permissions"
```

## Escalation Conditions

- CloudWatch metrics exist but Lookout for Metrics cannot read them despite correct permissions
- Cross-account CloudWatch access requires organization-level configuration changes
- Metric publishing frequency is incompatible with available detector frequencies
- CloudWatch metric data has gaps that cannot be resolved at the source

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — metric names and dimensions only
- `aws cloudwatch get-metric-statistics`: MEDIUM — actual metric values
- `aws lookoutmetrics describe-metric-set`: MEDIUM — contains CloudWatch configuration details
- CloudWatch metric data: MEDIUM — operational metrics that may reveal infrastructure details

## Prohibited Actions

- NEVER grant `cloudwatch:*` permissions — use only required read actions
- NEVER modify CloudWatch metrics or alarms during troubleshooting without approval
- NEVER assume metric names are case-insensitive — CloudWatch is case-sensitive
- NEVER configure detectors against CloudWatch metrics that publish infrequently (less than detector frequency)

## Phase 3 — Rollback

1. If metric set CloudWatch configuration was changed, revert to previous settings (requires detector deactivation)
2. If IAM permissions were broadened, revert to least-privilege after resolution
3. If CloudWatch metric filters were created for testing, clean up test filters
4. Document all configuration changes
5. Verify data ingestion resumes by checking data quality metrics

## Common Issues

- symptoms: "CloudWatch data source returns no data points"
  diagnosis: "Metric namespace, name, or dimensions do not match exactly. CloudWatch is case-sensitive."
  resolution: "Verify exact metric details: `aws cloudwatch list-metrics --namespace <ns>`. Update metric set to match."

- symptoms: "Intermittent data gaps from CloudWatch source"
  diagnosis: "Metric publishing frequency is lower than detector evaluation frequency."
  resolution: "Align detector frequency with metric publishing cadence. Use PT1H or P1D for infrequently published metrics."

## Output Format

```yaml
root_cause: "cloudwatch_data_source — <specific_cause>"
evidence:
  - type: cloudwatch_configuration
    content: "<metric details, namespace, dimensions, and data availability>"
severity: MEDIUM
mitigation:
  immediate: "Fix CloudWatch metric configuration"
  long_term: "Implement metric existence validation before detector configuration"
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
