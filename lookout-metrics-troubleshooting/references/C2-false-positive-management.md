---
title: "C2 — False Positive Management"
description: "Diagnose and reduce false positive anomaly detections"
status: active
severity: MEDIUM
triggers: ["too many false positives", "false anomalies", "noise in detection", "incorrect anomaly alerts"]
owner: devops-agent
objective: "Reduce false positive rate while maintaining detection of genuine anomalies"
context: "False positives are expected in anomaly detection. The sensitivity threshold, data quality, and seasonal patterns all affect false positive rates. Tuning requires iterative adjustment based on operational feedback."
---

## Phase 1 — Triage

MUST:
- List recent anomaly groups with current sensitivity: `aws lookoutmetrics list-anomaly-group-summaries --anomaly-detector-arn <arn> --sensitivity-threshold 50`
- Review anomaly group details for false positives: `aws lookoutmetrics get-anomaly-group --anomaly-detector-arn <arn> --anomaly-group-id <id>`
- Check current sensitivity threshold in detector configuration

SHOULD:
- Compare anomaly counts at different sensitivity thresholds (25, 50, 75)
- Identify patterns in false positives (time of day, day of week, specific dimensions)
- Check data quality for irregularities causing false detections

MAY:
- Review feedback history if anomaly feedback has been provided
- Analyze whether seasonal patterns are causing expected variations to be flagged

## Phase 2 — Remediate

MUST:
- Provide feedback on false positives: `aws lookoutmetrics put-feedback --anomaly-detector-arn <arn> --anomaly-group-time-series-feedback '[{"TimeSeriesId":"<id>","IsAnomaly":false}]'`
- Lower sensitivity threshold if false positive rate is unacceptable

SHOULD:
- Establish a regular feedback loop to improve model accuracy over time
- Review dimension configuration — overly granular dimensions increase false positives
- Check if data preprocessing can smooth expected variations

MAY:
- Create alert filters to suppress known false positive patterns
- Adjust metric set dimensions to reduce noise from low-volume segments

## Safety Ratings

```
safety_ratings:
  - "list-anomaly-group-summaries: GREEN — read-only anomaly listing"
  - "get-anomaly-group: GREEN — read-only anomaly details"
  - "put-feedback: YELLOW — modifies model learning, affects future detection"
  - "Sensitivity adjustment: YELLOW — changes detection behavior"
```

## Escalation Conditions

- False positive rate exceeds 50% despite sensitivity tuning and feedback
- Feedback does not improve detection accuracy after multiple iterations
- False positives are causing alert fatigue and genuine anomalies are being ignored
- Business stakeholders lose confidence in the anomaly detection system

## Data Sensitivity

- `aws lookoutmetrics get-anomaly-group`: MEDIUM — reveals business metric values and anomaly patterns
- `aws lookoutmetrics put-feedback`: LOW — sends boolean feedback on anomaly validity
- Anomaly group time series data: MEDIUM — contains actual business metric values
- Sensitivity configuration: LOW — operational parameter

## Prohibited Actions

- NEVER set sensitivity to 0 or near-zero to eliminate false positives — this disables meaningful detection
- NEVER provide incorrect feedback (marking genuine anomalies as not anomalous) to reduce alerts
- NEVER delete the detector to "start fresh" as a false positive fix — feedback and tuning are the correct approach
- NEVER ignore false positives without providing feedback — the model needs feedback to improve

## Phase 3 — Rollback

1. If sensitivity was lowered too aggressively, increase it incrementally
2. If incorrect feedback was provided, there is no direct undo — provide corrective feedback on future similar anomalies
3. If dimension changes caused detection gaps, revert metric set configuration (requires deactivation)
4. Document all tuning changes and their impact on detection rates
5. Maintain a log of confirmed false positives for pattern analysis

## Common Issues

- symptoms: "Anomaly alerts every Monday morning for expected weekly patterns"
  diagnosis: "Detector has not learned weekly seasonality or insufficient training data for weekly patterns."
  resolution: "Provide feedback marking Monday patterns as not anomalous. Ensure at least 4 weeks of training data."

- symptoms: "Low-volume dimensions generate constant false positives"
  diagnosis: "Dimensions with very few data points have high variance, triggering frequent anomalies."
  resolution: "Consider removing low-volume dimensions or aggregating them. Lower sensitivity for those metric sets."

## Output Format

```yaml
root_cause: "false_positive — <specific_cause>"
evidence:
  - type: false_positive_pattern
    content: "<anomaly group details and feedback history>"
severity: MEDIUM
mitigation:
  immediate: "Adjust sensitivity and provide feedback"
  long_term: "Establish regular feedback loop and dimension optimization"
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
