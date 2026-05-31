---
title: "E2 — Anomaly Detection"
description: "Troubleshoot CloudTrail Insights anomaly detection accuracy"
status: active
severity: LOW
triggers:
  - "false anomaly"
  - "missed anomaly"
  - "Insights accuracy"
  - "anomaly tuning"
owner: devops-agent
objective: "Improve CloudTrail Insights anomaly detection accuracy"
context: "Insights uses statistical analysis to detect anomalies in API call rates and error rates. False positives can occur during legitimate traffic spikes. False negatives can occur if anomalies are within normal variance."
---

## Phase 1 — Triage

MUST:
- Review recent Insights events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=insights.cloudtrail.amazonaws.com --max-results 20`
- Check Insights event details for the anomalous API and baseline comparison
- Verify the anomaly corresponds to actual unusual activity

SHOULD:
- Correlate Insights events with known deployments or maintenance windows
- Check if the baseline has stabilized (7+ days of consistent activity)
- Review API call patterns during the anomaly period

MAY:
- Compare Insights events with CloudWatch metrics
- Check if multiple Insights events correlate to a single root cause

## Phase 2 — Remediate

MUST:
- Investigate each Insights event to determine if it's a true anomaly
- Document known legitimate activity patterns that trigger false positives
- Ensure baseline has sufficient history for accurate detection

SHOULD:
- Set up EventBridge rules to route Insights events for investigation
- Create runbooks for common Insights event types
- Correlate with other monitoring data

MAY:
- Use CloudTrail Lake to analyze historical patterns
- Create automated triage for common false positive patterns

## Common Issues

- symptoms: "Insights fires during every deployment"
  diagnosis: "Deployments cause API call spikes that exceed baseline."
  resolution: "Document as expected. Create automation to auto-close deployment-related Insights."

- symptoms: "Actual attack not detected by Insights"
  diagnosis: "Attack API volume within normal variance or uses data events."
  resolution: "Insights only monitors management events. Use GuardDuty for threat detection."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate Insights events | GREEN | Read-only analysis |
| Set up EventBridge rules for routing | GREEN | Monitoring-only addition |
| Document known patterns | GREEN | Documentation only |

## Escalation Conditions

- Insights event correlates with potential security incident — escalate immediately to security team
- Persistent anomalies not explained by known activity — escalate to security operations
- Insights accuracy concerns affecting security posture — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail lookup-events` (Insights) | HIGH | Exposes anomaly detection results and API patterns |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** auto-suppress Insights events without human review

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created EventBridge rules | Delete the rules: `aws events delete-rule` |
| No state-changing actions in anomaly analysis | No rollback needed |

## Output Format

```yaml
root_cause: "anomaly_detection — <specific_cause>"
evidence:
  - type: insights_events
    content: "<recent Insights events>"
  - type: correlation
    content: "<correlation with known activity>"
severity: LOW
mitigation:
  immediate: "Investigate and classify Insights events"
  long_term: "Build automated triage for Insights events"
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
