---
title: "C1 — Anomaly Detection"
description: "Diagnose cost anomaly detection configuration issues"
status: active
severity: MEDIUM
triggers:
  - "anomaly detection not working"
  - "no anomaly alerts"
  - "cost anomaly setup"
  - "anomaly monitor"
owner: devops-agent
objective: "Resolve cost anomaly detection configuration and alerting issues"
context: "Cost Anomaly Detection uses ML to identify unusual spending. It requires 2+ weeks of historical data. Monitors can track by service, linked account, cost category, or cost allocation tag. Alerts are sent via SNS or email."
---

## Phase 1 — Triage

MUST:
- List anomaly monitors: `aws ce get-anomaly-monitors`
- Check anomaly subscriptions: `aws ce get-anomaly-subscriptions`
- Verify monitor has sufficient historical data (2+ weeks)
- Check for recent anomalies: `aws ce get-anomalies --date-interval StartDate=<start>,EndDate=<end>`

SHOULD:
- Verify SNS topic for anomaly alerts has correct permissions
- Check anomaly threshold configuration
- Review monitor type (service, account, tag, cost category)

MAY:
- Check if anomaly detection is available in the account's region
- Review historical anomaly detection accuracy

## Phase 2 — Remediate

MUST:
- Create anomaly monitor if none exists: `aws ce create-anomaly-monitor --anomaly-monitor '{"MonitorName":"<name>","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'`
- Create subscription for alerts: `aws ce create-anomaly-subscription --anomaly-subscription '{"SubscriptionName":"<name>","MonitorArnList":["<arn>"],"Subscribers":[{"Address":"<email>","Type":"EMAIL"}],"Frequency":"DAILY"}'`
- Wait 2+ weeks for baseline to establish

SHOULD:
- Set appropriate dollar threshold to reduce noise
- Configure alert frequency (immediate, daily, weekly)
- Create separate monitors for different cost dimensions

MAY:
- Integrate anomaly alerts with incident management systems
- Set up custom anomaly processing via SNS + Lambda

## Common Issues

- symptoms: "No anomalies detected despite cost spike"
  diagnosis: "Monitor too new (insufficient baseline) or threshold too high."
  resolution: "Wait for 2+ weeks of data. Lower the dollar threshold."

- symptoms: "Anomaly alerts not being received"
  diagnosis: "Subscription not created or SNS topic permissions missing."
  resolution: "Create subscription with correct email/SNS. Verify SNS permissions."

- symptoms: "Monitor shows no data"
  diagnosis: "Monitor type doesn't match spending patterns or account is new."
  resolution: "Verify monitor type. Ensure account has 2+ weeks of spending history."

## Output Format

```yaml
root_cause: "anomaly_detection — <specific_cause>"
evidence:
  - type: monitor_config
    content: "<anomaly monitor configuration>"
  - type: subscription_config
    content: "<alert subscription details>"
severity: MEDIUM
mitigation:
  immediate: "Create/fix monitors and subscriptions"
  long_term: "Tune thresholds based on spending patterns"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Anomaly data reveals spending patterns and deviations |
| Blast Radius | LOW — Anomaly detection is read-only monitoring |
| Rollback Complexity | LOW — Monitors and subscriptions can be updated or deleted |
| Compliance Scope | MEDIUM — Anomaly detection supports financial governance |

## Escalation Conditions

- Escalate to security if cost anomalies indicate unauthorized resource provisioning
- Escalate to finance if anomaly detection is not functioning and spending is unmonitored
- Escalate to AWS Support if anomaly detection fails to identify obvious cost spikes

## Data Sensitivity

- **Anomaly details** reveal which services had unusual spending — treat as financial data
- **SNS topic configurations** for alerts are operational data

## PROHIBITED Actions

- Do NOT delete anomaly monitors without setting up alternative cost monitoring
- Do NOT ignore cost anomalies — they may indicate security incidents
- Do NOT set thresholds so high that meaningful anomalies are missed

## Phase 3 — Rollback

- Restore deleted monitors: `aws ce create-anomaly-monitor --anomaly-monitor <config>`
- Restore deleted subscriptions: `aws ce create-anomaly-subscription --anomaly-subscription <config>`
- Note: new monitors need 2+ weeks to rebuild baselines
- Document any monitoring gaps for audit purposes

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
