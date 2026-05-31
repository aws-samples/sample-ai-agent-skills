---
title: "E1 — Insights Events"
description: "Diagnose CloudTrail Insights event issues"
status: active
severity: MEDIUM
triggers:
  - "Insights not working"
  - "no Insights events"
  - "anomaly not detected"
  - "Insights configuration"
owner: devops-agent
objective: "Configure and troubleshoot CloudTrail Insights for anomaly detection"
context: "CloudTrail Insights detects unusual API activity by analyzing management event patterns. It requires a 7-day baseline period. Insights can detect anomalies in API call rates and error rates."
---

## Phase 1 — Triage

MUST:
- Check Insights configuration: `aws cloudtrail get-insight-selectors --trail-name <trail-name>`
- Verify trail is logging management events: `aws cloudtrail get-trail-status --name <trail-name>`
- Check if 7-day baseline period has passed
- Search for Insights events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=insights.cloudtrail.amazonaws.com --max-results 10`

SHOULD:
- Verify Insights types enabled (ApiCallRateInsight, ApiErrorRateInsight)
- Check if sufficient management event volume exists for baseline
- Review recent Insights events for accuracy

MAY:
- Check CloudTrail Lake for Insights events
- Review Insights event delivery to S3

## Phase 2 — Remediate

MUST:
- Enable Insights: `aws cloudtrail put-insight-selectors --trail-name <trail-name> --insight-selectors '[{"InsightType":"ApiCallRateInsight"},{"InsightType":"ApiErrorRateInsight"}]'`
- Wait 7 days for baseline establishment
- Ensure trail logs management events

SHOULD:
- Enable both ApiCallRateInsight and ApiErrorRateInsight
- Monitor Insights events after baseline period
- Set up EventBridge rules for Insights event notifications

MAY:
- Configure Insights delivery to CloudTrail Lake
- Create dashboards for Insights trends

## Common Issues

- symptoms: "Insights enabled but no events generated"
  diagnosis: "Baseline period not complete or no anomalies detected."
  resolution: "Wait 7 days for baseline. Insights only fires on actual anomalies."

- symptoms: "Too many Insights events (noisy)"
  diagnosis: "Normal activity patterns triggering false anomalies."
  resolution: "Review and tune. Insights will adjust baseline over time."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Insights | GREEN | Additive feature, does not affect existing logging |
| Wait for baseline | GREEN | No action required, passive monitoring |
| Set up EventBridge rules | GREEN | Monitoring-only addition |

## Escalation Conditions

- Insights detecting actual anomalous activity — escalate to security team for investigation
- Insights generating excessive false positives — escalate to operations team for tuning
- Insights not detecting known anomalies — escalate to security team for supplementary detection

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail get-insight-selectors` | HIGH | Exposes Insights configuration |
| `aws cloudtrail lookup-events` (Insights) | HIGH | Exposes anomaly detection results |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** dismiss Insights events without investigation

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled Insights | Disable Insights: `aws cloudtrail put-insight-selectors --trail-name <name> --insight-selectors '[]'` |
| Created EventBridge rules | Delete the rules: `aws events delete-rule` |

## Output Format

```yaml
root_cause: "insights — <specific_cause>"
evidence:
  - type: insight_selectors
    content: "<Insights configuration>"
  - type: baseline_status
    content: "<baseline establishment status>"
severity: MEDIUM
mitigation:
  immediate: "Enable Insights and wait for baseline"
  long_term: "Monitor and tune Insights detection"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
