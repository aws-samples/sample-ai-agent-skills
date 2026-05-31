---
title: "H2 — Bot Analytics Issues"
description: "Diagnose bot analytics and metrics problems"
status: active
severity: LOW
triggers:
  - "analytics not showing"
  - "metrics missing"
  - "bot performance data"
  - "utterance statistics"
owner: devops-agent
objective: "Identify and resolve bot analytics issues"
context: "Lex provides CloudWatch metrics for bot performance. Analytics include intent recognition rates, missed utterances, and session metrics. Metrics may have propagation delay."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch metrics: `aws cloudwatch list-metrics --namespace AWS/Lex --dimensions Name=BotId,Value=<bot-id>`
- Get recognition metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Lex --metric-name RuntimeSuccessfulRequestCount --dimensions Name=BotId,Value=<bot-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify bot has received traffic (metrics only appear after usage)
- Check Lex V2 analytics: `aws lexv2-models list-utterance-analytics --bot-id <bot-id> --start-date-time <start> --end-date-time <end>`

SHOULD:
- Check for missed utterance patterns
- Review intent-level metrics
- Compare metrics across time periods

MAY:
- Set up CloudWatch dashboards for bot metrics
- Configure alarms for anomalous patterns

## Phase 2 — Remediate

MUST:
- Ensure bot is receiving traffic (metrics require actual usage)
- Allow propagation time for metrics (up to 15 minutes)

SHOULD:
- Create CloudWatch dashboard for key bot metrics
- Set up alarms for high missed utterance rates

MAY:
- Implement custom analytics pipeline with conversation logs

## Common Issues

- symptoms: "No metrics in CloudWatch for bot"
  diagnosis: "Bot hasn't received any traffic or wrong dimension values."
  resolution: "Send test traffic. Verify BotId dimension matches."

- symptoms: "Analytics show high error rate"
  diagnosis: "Lambda fulfillment failures or bot configuration issues."
  resolution: "Check Lambda logs. Review bot configuration."

## Output Format

```yaml
root_cause: "analytics — <specific_cause>"
evidence:
  - type: cloudwatch_metrics
    content: "<bot metrics>"
  - type: utterance_analytics
    content: "<utterance analytics>"
severity: LOW
mitigation:
  immediate: "Verify metrics configuration"
  long_term: "Implement comprehensive bot analytics dashboard"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or overwrite a published bot alias that is actively serving production traffic
- **NEVER** modify intent or slot configurations on a live bot version without creating a new draft version first
- **NEVER** remove Lambda fulfillment hooks from intents without verifying downstream dependencies
- **NEVER** disable conversation logs on bots subject to compliance or audit requirements
- **NEVER** migrate a V1 bot to V2 in-place without maintaining the V1 bot as a fallback

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
