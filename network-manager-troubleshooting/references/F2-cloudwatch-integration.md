---
title: "F2 — CloudWatch Integration"
description: "Diagnose CloudWatch integration issues with Network Manager metrics and events"
status: active
severity: MEDIUM
triggers: ["cloudwatch integration error", "network manager metrics missing", "cloudwatch events not firing", "network manager alarms not working"]
owner: devops-agent
objective: "Resolve CloudWatch integration issues for Network Manager monitoring"
context: "Network Manager publishes metrics to CloudWatch and topology change events to EventBridge. Integration requires proper IAM permissions and regional configuration."
---

## Phase 1 — Triage

MUST:
- List available Network Manager metrics: `aws cloudwatch list-metrics --namespace AWS/NetworkManager`
- Check EventBridge rules for Network Manager events: `aws events list-rules --name-prefix networkmanager`
- Verify IAM permissions for CloudWatch and EventBridge access

SHOULD:
- Check if metrics are published in the correct region (us-west-2 for global network metrics)
- Verify EventBridge rule patterns match Network Manager event types
- Review CloudWatch alarm configurations for Network Manager metrics

MAY:
- Check CloudWatch Logs for Network Manager event delivery
- Verify SNS topic subscriptions for alarm notifications

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for cloudwatch:PutMetricData and events:PutEvents
- Ensure CloudWatch queries target the correct region for Network Manager metrics
- Fix EventBridge rule patterns to match Network Manager event schema

SHOULD:
- Create CloudWatch alarms for critical Network Manager metrics
- Configure EventBridge rules for topology change notifications

MAY:
- Set up CloudWatch dashboards for Network Manager overview
- Implement cross-region metric aggregation

## Common Issues

- symptoms: "No Network Manager metrics in CloudWatch"
  diagnosis: "Querying the wrong region. Network Manager global metrics are published to us-west-2."
  resolution: "Query CloudWatch metrics in us-west-2 for global network metrics."

- symptoms: "EventBridge rules not triggering for Network Manager events"
  diagnosis: "EventBridge rule pattern does not match Network Manager event schema."
  resolution: "Update the rule pattern to match source: aws.networkmanager and the correct detail-type."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
  - "events list-rules: GREEN — read-only rule listing"
  - "cloudwatch put-metric-alarm: YELLOW — creates monitoring alarm"
  - "events put-rule: YELLOW — creates event routing rule"
```

## Escalation Conditions

- CloudWatch metrics completely absent for Network Manager across all regions
- EventBridge events not delivered despite correct rule configuration
- CloudWatch alarms firing false positives causing alert fatigue
- Cross-region metric aggregation failing for global network monitoring

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — returns metric names only
- `aws events list-rules`: LOW — returns rule metadata
- `aws cloudwatch get-metric-data`: MEDIUM — reveals network performance data
- `aws events describe-rule`: LOW — returns rule configuration

## Prohibited Actions

- NEVER delete existing CloudWatch alarms without confirming they are not used by other teams
- NEVER modify EventBridge rules that feed into incident management systems
- NEVER assume metrics are missing because the service is down — check the correct region first
- NEVER create high-frequency alarms that could cause alert fatigue

## Phase 3 — Rollback

1. If CloudWatch alarms were modified, revert to previous configuration
2. If EventBridge rules were changed, restore previous rule patterns
3. Verify alarm and event delivery is functioning as before
4. Confirm no monitoring gaps were introduced
5. Document changes and notify the monitoring team

## Output Format

```yaml
root_cause: "cloudwatch_integration — <specific_cause>"
evidence:
  - type: integration_state
    content: "<CloudWatch or EventBridge details>"
severity: MEDIUM
mitigation:
  immediate: "Fix CloudWatch integration"
  long_term: "Implement comprehensive Network Manager monitoring"
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
