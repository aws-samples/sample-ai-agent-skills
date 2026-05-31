---
title: "D2 — Metrics Issues"
description: "Diagnose Incident Manager metrics integration problems including CloudWatch metrics display"
status: active
severity: MEDIUM
triggers:
  - "incident metrics"
  - "metrics not showing"
  - "cloudwatch metrics incident"
  - "metric dashboard"
owner: devops-agent
objective: "Resolve metrics integration issues in incident records"
context: "Response plans can include CloudWatch metrics that are displayed in the incident dashboard. Common issues include wrong metric configuration, missing CloudWatch permissions, and metrics not appearing in the incident view."
---

## Phase 1 — Triage

MUST:
- Check response plan metric configuration: `aws ssm-incidents get-response-plan --arn <plan-arn>`
- Verify CloudWatch metrics exist: `aws cloudwatch list-metrics --namespace <namespace> --metric-name <metric>`
- Check IAM permissions for CloudWatch metric access
- Verify the incident record has metrics associated

SHOULD:
- Check if metrics are in the correct region
- Verify metric dimensions match the response plan configuration
- Check CloudWatch metric data availability for the incident time range

MAY:
- Test metric query: `aws cloudwatch get-metric-statistics --namespace <namespace> --metric-name <metric> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check for custom metrics vs standard metrics

## Phase 2 — Remediate

MUST:
- Fix metric configuration in response plan (namespace, metric name, dimensions)
- Grant CloudWatch read permissions to the Incident Manager role
- Verify metrics exist in the correct region

SHOULD:
- Use specific metric dimensions for targeted monitoring
- Configure multiple metrics for comprehensive incident visibility

## Output Format

```yaml
root_cause: "metrics — <specific_cause>"
evidence:
  - type: metric_config
    content: "<response plan metric configuration>"
  - type: metric_data
    content: "<CloudWatch metric availability>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric configuration or permissions"
  long_term: "Standardize metric configuration across response plans"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete response plans that are actively referenced by CloudWatch alarms
- Never modify escalation plans during active incidents
- Never remove contact channels without confirming alternative notification paths exist
- Never delete incident records or post-incident analysis data
- Never disable automation runbooks tied to active response plans without backup procedures

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous response plan configuration if incident detection breaks
- Revert escalation plan changes if notifications fail to reach responders
- Re-enable previous contact channel settings if engagement plans stop working

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
