---
title: "D2 — Metric Policy Errors"
description: "Diagnose MediaStore metric policy configuration errors"
status: active
severity: MEDIUM
triggers: ["metric policy error", "no cloudwatch metrics", "metrics not showing", "metric policy failed"]
owner: devops-agent
objective: "Resolve MediaStore metric policy configuration issues for CloudWatch monitoring"
context: "Metric policies control which CloudWatch metrics are published and at what granularity. Without a metric policy, only basic container-level metrics are available."
---

## Phase 1 — Triage

MUST:
- Get current metric policy: `aws mediastore get-metric-policy --container-name <name>`
- Check if metrics are being published: `aws cloudwatch list-metrics --namespace AWS/MediaStore --dimensions Name=ContainerName,Value=<name>`
- Verify IAM permissions for mediastore:PutMetricPolicy

SHOULD:
- Check for metric policy syntax errors in the JSON
- Verify the metric policy rules target valid path prefixes

MAY:
- Compare metrics availability with and without the metric policy

## Phase 2 — Remediate

MUST:
- Set metric policy: `aws mediastore put-metric-policy --container-name <name> --metric-policy file://metric-policy.json`
- Set ContainerLevelMetrics to ENABLED in the policy
- Verify metrics appear in CloudWatch after applying

SHOULD:
- Add path-level metric rules for important content paths
- Set up CloudWatch dashboards for MediaStore metrics

MAY:
- Create CloudWatch alarms for key metrics (4xxErrorCount, 5xxErrorCount)

## Safety Ratings

```
safety_ratings:
  - "mediastore get-metric-policy: GREEN — read-only policy retrieval"
  - "mediastore put-metric-policy: GREEN — enables metrics, non-destructive"
  - "mediastore delete-metric-policy: YELLOW — stops detailed metrics"
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
```

## Escalation Conditions

- Metric policy is set but no metrics appear in CloudWatch after 15 minutes
- Path-level metrics are needed but the number of rules exceeds the limit
- Metric policy causes unexpected CloudWatch costs
- Need custom metrics not available through the metric policy

## Data Sensitivity

- Metric policy document: LOW — contains metric rules and path prefixes
- CloudWatch metrics: LOW — numeric performance data
- Path prefixes in rules: LOW — content organization structure
- CloudWatch dashboard configuration: LOW — visualization settings

## Prohibited Actions

- NEVER delete the metric policy during an active incident — you lose visibility
- NEVER create excessive path-level rules — each rule generates additional metrics and costs
- NEVER ignore missing metrics — they indicate monitoring gaps
- NEVER assume container-level metrics are sufficient for production monitoring

## Phase 3 — Rollback

1. Restore previous metric policy: `aws mediastore put-metric-policy --container-name <name> --metric-policy file://previous-policy.json`
2. If metric policy was deleted, re-apply
3. Verify CloudWatch metrics resume after restoration
4. Check CloudWatch alarms are still functional
5. Document the metric policy change

## Common Issues

- symptoms: "No path-level metrics despite metric policy being set"
  diagnosis: "MetricPolicyRules path prefix does not match actual object paths."
  resolution: "Verify path prefixes in rules match the container's object path structure."

- symptoms: "ContainerLevelMetrics show but path-level metrics do not"
  diagnosis: "MetricPolicyRules array is empty or rules are misconfigured."
  resolution: "Add rules with correct ObjectGroup and ObjectGroupName for each path prefix."

## Output Format

```yaml
root_cause: "metric_policy — <specific_cause>"
evidence:
  - type: metric_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric policy configuration"
  long_term: "Include metric policy in IaC and set up dashboards"
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
