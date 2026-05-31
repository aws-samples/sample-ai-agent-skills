---
title: "E3 — CloudWatch WAF Metrics Issues"
description: "Diagnose CloudWatch metric anomalies and monitoring gaps for WAF"
status: active
severity: MEDIUM
triggers:
  - "CloudWatch metrics"
  - "WAF metrics"
  - "blocked requests metric"
  - "allowed requests metric"
  - "WAF monitoring"
owner: devops-agent
objective: "Identify and resolve CloudWatch metric issues for WAF monitoring and alerting"
context: "WAF publishes metrics to CloudWatch under the AWS/WAFV2 namespace. Key metrics include AllowedRequests, BlockedRequests, CountedRequests, and PassedRequests. Metrics are available per web ACL, per rule, and per label. Dimensions include WebACL, Rule, and Region. Metrics have a 1-minute minimum period. CloudWatch alarms can be set on these metrics for automated alerting."
---

## Phase 1 — Triage

MUST:
- Check available metrics: `aws cloudwatch list-metrics --namespace AWS/WAFV2 --dimensions Name=WebACL,Value=<acl-name>`
- Get blocked request metrics: `aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=ALL Name=Region,Value=<region> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Get allowed request metrics: `aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name AllowedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=ALL Name=Region,Value=<region> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check per-rule metrics to identify which rules are most active

SHOULD:
- Compare current metrics with historical baselines to identify anomalies
- Check for sudden spikes in blocked or allowed requests
- Verify CloudWatch alarms are configured for critical thresholds

MAY:
- Set up CloudWatch dashboards for WAF metrics visualization
- Create composite alarms combining WAF metrics with application metrics

## Phase 2 — Remediate

MUST:
- Create CloudWatch alarms for abnormal block rates: `aws cloudwatch put-metric-alarm --alarm-name waf-high-block-rate --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=ALL Name=Region,Value=<region> --statistic Sum --period 300 --threshold <threshold> --comparison-operator GreaterThanThreshold --evaluation-periods 2 --alarm-actions <sns-topic-arn>`
- Investigate sudden metric changes by correlating with rule changes and traffic patterns

SHOULD:
- Set up alarms for both high block rates (potential attack) and low block rates (potential rule failure)
- Create per-rule metric dashboards for visibility
- Configure SNS notifications for alarm state changes

MAY:
- Implement automated response to metric alarms (e.g., Lambda to add IPs to block list)
- Use CloudWatch Anomaly Detection for dynamic thresholds

## Common Issues

- symptoms: "No WAF metrics appearing in CloudWatch"
  diagnosis: "Web ACL is not associated with any resource, or the region dimension is incorrect."
  resolution: "Verify the web ACL is associated. Use the correct region (us-east-1 for CLOUDFRONT scope)."

- symptoms: "Metrics show zero blocked requests despite active rules"
  diagnosis: "All rules are in Count mode (OverrideAction: Count on managed groups)."
  resolution: "Count mode requests appear in CountedRequests, not BlockedRequests. Check CountedRequests metric."

- symptoms: "Sudden spike in blocked requests"
  diagnosis: "New rule added, rule group version updated, or actual attack traffic."
  resolution: "Check CloudTrail for recent web ACL changes. Review sampled requests for the blocking rule."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create CloudWatch alarms | YELLOW | New resource; can be deleted |
| Investigate metric anomalies | GREEN | Read-only analysis |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Sudden metric spikes indicating potential attacks

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| CloudWatch metrics | LOW | Aggregate request counts |
| `list-metrics` | LOW | Metric names and dimensions |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| CloudWatch alarm creation | Delete alarm via `delete-alarms` |

## Output Format

```yaml
root_cause: "cloudwatch_metrics — <specific_cause>"
evidence:
  - type: metric_data
    content: "<metric values over time>"
  - type: alarm_status
    content: "<alarm state and configuration>"
  - type: rule_metrics
    content: "<per-rule metric breakdown>"
severity: MEDIUM
mitigation:
  immediate: "Investigate metric anomaly and correlate with rule changes"
  long_term: "Implement comprehensive CloudWatch monitoring with alarms and dashboards"
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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
