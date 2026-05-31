---
title: "C2 — Metric Collection Failures"
description: "Diagnose CloudWatch Application Signals metric collection failures"
status: active
severity: HIGH
triggers: ["metrics missing", "no metrics", "metric collection failed", "AWS/ApplicationSignals empty"]
owner: devops-agent
objective: "Resolve metric collection and publishing failures for Application Signals"
context: "Application Signals metrics are published to the AWS/ApplicationSignals namespace by the CloudWatch agent."
---

## Phase 1 — Triage

MUST:
- Check for metrics in namespace: `aws cloudwatch list-metrics --namespace 'AWS/ApplicationSignals'`
- Verify CloudWatch agent is running and configured for Application Signals
- Check CloudWatch agent logs for metric publishing errors

SHOULD:
- Verify ADOT agent is generating metrics (check ADOT agent logs)
- Check CloudWatch agent configuration file for application_signals section
- Verify network connectivity to CloudWatch endpoints

MAY:
- Check CloudWatch agent metrics for self-monitoring: `aws cloudwatch get-metric-statistics --namespace CWAgent --metric-name mem_used_percent`
- Review VPC endpoint configuration for CloudWatch

## Phase 2 — Remediate

MUST:
- Fix CloudWatch agent configuration to include application_signals section
- Ensure ADOT agent OTLP exporter points to the CloudWatch agent endpoint
- Verify IAM permissions include cloudwatch:PutMetricData

SHOULD:
- Restart CloudWatch agent after configuration changes
- Verify metrics appear within 5 minutes of fix

MAY:
- Enable CloudWatch agent debug logging for detailed diagnostics
- Configure metric aggregation intervals

## Common Issues

- symptoms: "No metrics in AWS/ApplicationSignals namespace"
  diagnosis: "CloudWatch agent configuration does not include application_signals section."
  resolution: "Add application_signals configuration to CloudWatch agent config. Restart the agent."

- symptoms: "Metrics appear for some services but not others"
  diagnosis: "Some services are not instrumented or ADOT agent is not running on those hosts."
  resolution: "Verify ADOT agent is deployed on all hosts. Check per-service instrumentation status."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only metric listing"
  - "CloudWatch agent log review: GREEN — read-only log analysis"
  - "CloudWatch agent restart: YELLOW — brief metric collection gap"
  - "CloudWatch agent configuration change: YELLOW — affects metric collection"
```

## Escalation Conditions

- Metric collection fails despite correct agent configuration
- CloudWatch agent crashes repeatedly when Application Signals is enabled
- Metric publishing blocked by VPC endpoint or network configuration
- High metric volume causes CloudWatch agent memory issues

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — returns metric names and dimensions
- CloudWatch agent logs: MEDIUM — may contain endpoint URLs and configuration
- CloudWatch agent configuration: MEDIUM — contains monitoring configuration
- Application metrics: MEDIUM — reveal service performance characteristics

## Prohibited Actions

- NEVER stop the CloudWatch agent on production instances without a maintenance window
- NEVER delete CloudWatch metrics or metric streams to troubleshoot
- NEVER use overly broad IAM permissions (cloudwatch:*) to fix metric publishing
- NEVER ignore metric collection gaps — they affect SLO evaluation

## Phase 3 — Rollback

1. Restore previous CloudWatch agent configuration if changes caused issues
2. Restart CloudWatch agent with the original configuration
3. Verify metric collection resumes within 5 minutes
4. Check SLO evaluations are not affected by the metric gap
5. Document the metric collection failure and resolution

## Output Format

```yaml
root_cause: "metric_collection — <specific_cause>"
evidence:
  - type: metric_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix agent configuration or connectivity"
  long_term: "Implement metric collection monitoring"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
