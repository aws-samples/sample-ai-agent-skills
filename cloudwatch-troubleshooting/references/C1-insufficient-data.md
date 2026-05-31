---
title: "C1 — INSUFFICIENT_DATA Alarm State"
description: "Diagnose alarms stuck in INSUFFICIENT_DATA state"
status: active
severity: MEDIUM
triggers:
  - "INSUFFICIENT_DATA"
  - "alarm no data"
  - "alarm state insufficient"
  - "alarm not evaluating"
owner: devops-agent
objective: "Identify why an alarm is in INSUFFICIENT_DATA state and determine if action is needed"
context: "INSUFFICIENT_DATA means CloudWatch received no metric data during the alarm's evaluation period. This is NOT an error — it is normal for idle resources. Common causes: resource not emitting data, wrong metric/dimensions, metric math with missing inputs, or the alarm was just created and hasn't evaluated yet."
---

## Phase 1 — Triage

MUST:
- Check alarm configuration: `aws cloudwatch describe-alarms --alarm-names <name>`
- Verify the metric has data: `aws cloudwatch get-metric-statistics --namespace <ns> --metric-name <metric> --dimensions Name=<key>,Value=<val> --start-time <iso> --end-time <iso> --period <alarm-period> --statistics <alarm-stat>`
- Check the alarm's TreatMissingData setting (missing, notBreaching, breaching, ignore)
- Verify namespace, metric name, and dimensions are correct

SHOULD:
- Check alarm state history: `aws cloudwatch describe-alarm-history --alarm-name <name> --history-item-type StateUpdate`
- Verify the resource is running and actively emitting the metric
- Check if the alarm uses metric math (missing input = missing result)

MAY:
- Compare with a known-working alarm on the same metric
- Check if the alarm was recently created (initial state is INSUFFICIENT_DATA)

## Phase 2 — Remediate

MUST:
- Fix metric namespace, name, or dimensions if incorrect
- Set TreatMissingData appropriately for the use case
- Verify the resource is emitting the expected metric

SHOULD:
- Use TreatMissingData=notBreaching for metrics that are expected to be sparse
- Use TreatMissingData=breaching if missing data should trigger the alarm
- Add FILL() to metric math expressions to handle missing inputs

MAY:
- Create a separate alarm to detect when the metric stops publishing
- Document expected alarm behavior for the operations team

## Common Issues

- symptoms: "Alarm stuck in INSUFFICIENT_DATA after creation"
  diagnosis: "Alarm needs at least one evaluation period of data before transitioning."
  resolution: "Wait for the evaluation period to pass. Verify the metric is publishing data."

- symptoms: "Alarm was OK but moved to INSUFFICIENT_DATA"
  diagnosis: "Resource stopped emitting the metric (instance stopped, Lambda not invoked)."
  resolution: "Check resource status. Set TreatMissingData based on desired behavior."

- symptoms: "Metric math alarm always INSUFFICIENT_DATA"
  diagnosis: "One or more input metrics have no data, causing the expression to return missing."
  resolution: "Use FILL(metric, value) in the expression to handle missing data."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix metric reference | GREEN | Configuration correction |
| Set TreatMissingData | YELLOW | Changes alarm behavior; may trigger or suppress alarms |
| Verify resource status | GREEN | Read-only check |

## Escalation Conditions

- Production alarm stuck in INSUFFICIENT_DATA — escalate to monitoring team
- TreatMissingData changes on critical alarms — escalate to alarm owner
- Resource stopped emitting metrics unexpectedly — escalate to resource owner

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
- **NEVER** set TreatMissingData=notBreaching on alarms where missing data indicates a problem

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Changed TreatMissingData | Restore previous TreatMissingData setting |
| Fixed metric reference | Restore previous metric configuration |

## Output Format

```yaml
root_cause: "insufficient_data — <specific_cause>"
evidence:
  - type: alarm_config
    content: "<alarm configuration>"
  - type: metric_data
    content: "<metric statistics for the evaluation period>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric reference, TreatMissingData setting, or verify resource status"
  long_term: "Set TreatMissingData appropriately for all alarms"
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
