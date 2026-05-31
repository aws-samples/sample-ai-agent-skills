---
title: "C2 — Alarm Not Triggering"
description: "Diagnose why an alarm is not entering ALARM state despite apparent threshold breach"
status: active
severity: HIGH
triggers:
  - "alarm not triggering"
  - "alarm not firing"
  - "threshold breached"
  - "alarm stays OK"
  - "alarm evaluation"
owner: devops-agent
objective: "Identify why an alarm remains in OK state when the user expects it to be in ALARM"
context: "Alarms not triggering is usually caused by misunderstanding the M-of-N evaluation logic (datapoints to alarm vs evaluation periods), incorrect statistic selection (Average vs Maximum), period misalignment, or the metric not actually breaching the threshold when evaluated correctly."
---

## Phase 1 — Triage

MUST:
- Check alarm configuration: `aws cloudwatch describe-alarms --alarm-names <name>`
- Note: Period, EvaluationPeriods, DatapointsToAlarm, Statistic, ComparisonOperator, Threshold
- Query the actual metric data at the alarm's period: `aws cloudwatch get-metric-statistics --namespace <ns> --metric-name <metric> --dimensions Name=<key>,Value=<val> --start-time <iso> --end-time <iso> --period <alarm-period> --statistics <alarm-stat>`
- Check alarm history for state transitions: `aws cloudwatch describe-alarm-history --alarm-name <name>`

SHOULD:
- Verify the statistic matches the use case (Average smooths spikes, Maximum catches them)
- Check if TreatMissingData is set to notBreaching (missing periods count as not breaching)
- Verify the comparison operator direction (GreaterThanThreshold vs GreaterThanOrEqualToThreshold)

MAY:
- Graph the metric in the console to visually confirm threshold breach
- Compare the alarm's evaluation window with the actual metric data

## Phase 2 — Remediate

MUST:
- Fix DatapointsToAlarm if too high relative to EvaluationPeriods
- Change Statistic to Maximum if trying to catch spikes (Average smooths them out)
- Fix ComparisonOperator if inverted

SHOULD:
- Reduce Period for faster detection (minimum 10 seconds for high-res, 60 seconds for standard)
- Set DatapointsToAlarm = 1 for immediate alerting on any breach
- Use anomaly detection alarms for metrics without a clear static threshold

MAY:
- Create a test metric to verify alarm evaluation behavior
- Document the M-of-N logic for the operations team

## Common Issues

- symptoms: "CPU spikes to 100% but alarm doesn't trigger"
  diagnosis: "Alarm uses Average statistic with 5-minute period. Average smooths the spike."
  resolution: "Change statistic to Maximum, or reduce the period to 1 minute."

- symptoms: "Alarm requires 3 of 3 datapoints but metric is sparse"
  diagnosis: "DatapointsToAlarm equals EvaluationPeriods, and missing data is treated as notBreaching."
  resolution: "Reduce DatapointsToAlarm (e.g., 2 of 3) or set TreatMissingData to breaching."

- symptoms: "Metric exceeds threshold but alarm stays OK"
  diagnosis: "ComparisonOperator is GreaterThanThreshold but metric equals the threshold exactly."
  resolution: "Change to GreaterThanOrEqualToThreshold."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix DatapointsToAlarm | YELLOW | Changes alarm sensitivity; may trigger more/fewer alarms |
| Change Statistic to Maximum | YELLOW | Changes what the alarm evaluates; may trigger immediately |
| Fix ComparisonOperator | YELLOW | Changes alarm threshold logic |
| Reduce Period | YELLOW | Increases alarm evaluation frequency |

## Escalation Conditions

- Production alarm not triggering during actual incidents — escalate immediately to operations team
- Alarm configuration changes on critical infrastructure — escalate to alarm owner
- M-of-N evaluation logic changes — escalate to monitoring team for review

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch describe-alarms` | MEDIUM | Exposes alarm configuration and thresholds |
| `aws cloudwatch describe-alarm-history` | MEDIUM | Exposes alarm state transitions |
| `aws cloudwatch get-metric-statistics` | LOW | Numeric metric data |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** change alarm configuration during an active incident without approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Changed DatapointsToAlarm | Restore previous value |
| Changed Statistic | Restore previous statistic (e.g., Average) |
| Changed ComparisonOperator | Restore previous operator |
| Changed Period | Restore previous period |

## Output Format

```yaml
root_cause: "alarm_not_triggering — <specific_cause>"
evidence:
  - type: alarm_config
    content: "<alarm configuration with evaluation settings>"
  - type: metric_data
    content: "<actual metric values during expected breach>"
severity: HIGH
mitigation:
  immediate: "Fix statistic, evaluation periods, datapoints to alarm, or comparison operator"
  long_term: "Review all alarms for correct M-of-N configuration"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
