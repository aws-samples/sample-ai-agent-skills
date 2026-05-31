---
title: "B2 — Detect Metric Anomalies"
description: "Diagnose metric anomaly detection and violation issues"
status: active
severity: HIGH
triggers: ["false positive violation", "missing violation", "metric not reported", "anomaly detection error", "threshold misconfigured"]
owner: devops-agent
objective: "Resolve metric anomaly detection issues including false positives, missing violations, and threshold tuning"
context: "Detect monitors device metrics against security profile behaviors. Violations are generated when metrics exceed thresholds. Both cloud-side and device-side metrics are supported."
---

## Phase 1 — Triage

MUST:
- Check active violations: `aws iot list-active-violations --thing-name <thing-name>`
- Review violation details: `aws iot list-violation-events --thing-name <thing-name> --start-time <start> --end-time <end>`
- Check security profile behaviors: `aws iot describe-security-profile --security-profile-name <name> --query 'behaviors'`

SHOULD:
- Verify device is reporting metrics (for device-side metrics)
- Compare metric values against configured thresholds
- Check if violation is from statistical or absolute threshold

MAY:
- Review metric trends in CloudWatch
- Check ML Detect model status if using ML-based detection

## Phase 2 — Remediate

MUST:
- Adjust thresholds for false positive violations (too strict)
- Investigate true positive violations for potential security issues
- Ensure devices report metrics in the correct format

SHOULD:
- Use statistical thresholds for metrics with natural variation
- Configure consecutive data points required before triggering violation

MAY:
- Switch from static thresholds to ML Detect for adaptive detection
- Add dimensions to scope monitoring to specific topics

## Safety Ratings

```
safety_ratings:
  - "iot list-active-violations: GREEN — read-only violation listing"
  - "iot list-violation-events: GREEN — read-only violation history"
  - "iot describe-security-profile: GREEN — read-only profile details"
  - "iot update-security-profile (thresholds): YELLOW — changes detection sensitivity"
```

## Escalation Conditions

- Persistent violations on multiple devices indicating potential fleet-wide compromise
- Metric values significantly exceed normal ranges suggesting active attack
- False positive rate is too high making real violations hard to identify
- Device-side metrics stop reporting across multiple devices

## Data Sensitivity

- `aws iot list-active-violations`: MEDIUM — reveals device security status
- `aws iot list-violation-events`: MEDIUM — contains device identifiers and metric values
- `security profile behaviors`: LOW — contains threshold configuration
- `device metric reports`: MEDIUM — contain device network and system information

## Prohibited Actions

- NEVER dismiss violations without investigation — they may indicate compromised devices
- NEVER raise all thresholds to eliminate violations without understanding root cause
- NEVER disable Detect monitoring to stop violation alerts
- NEVER share device metric data containing IP addresses or connection details publicly

## Phase 3 — Rollback

1. If thresholds were changed incorrectly, restore previous values
2. If violations were cleared incorrectly, re-enable monitoring and wait for new data
3. Verify detection is working by checking for expected violations on test devices
4. Restore security profile to previous configuration if changes caused issues
5. Document threshold changes and their justification

## Common Issues

- symptoms: "Excessive false positive violations — devices triggering violations during normal operation"
  diagnosis: "Thresholds too strict for the device fleet's normal behavior."
  resolution: "Analyze metric baselines and adjust thresholds. Consider ML Detect for adaptive thresholds."

## Output Format

```yaml
root_cause: "metric_anomaly — <specific_cause>"
evidence:
  - type: violation_details
    content: "<violation metric, value, and threshold>"
severity: HIGH
mitigation:
  immediate: "Investigate violation and adjust thresholds"
  long_term: "Implement ML Detect for adaptive anomaly detection"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
