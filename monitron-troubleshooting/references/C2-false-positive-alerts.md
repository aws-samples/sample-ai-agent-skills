---
title: "C2 — False Positive Alerts"
description: "Diagnose and reduce false positive anomaly alerts"
status: active
severity: MEDIUM
triggers: ["false positive", "false alarm", "too many alerts", "alert fatigue", "incorrect anomaly"]
owner: devops-agent
objective: "Reduce false positive alerts and improve detection accuracy"
context: "False positives occur when the ML model flags normal operating conditions as anomalies. Common causes include sensor mounting issues, environmental changes, and normal operational variations."
---

## Phase 1 — Triage

MUST:
- Verify the alert corresponds to actual equipment inspection results
- Check sensor mounting position and orientation
- Confirm the 14-day learning period was completed under normal operating conditions

SHOULD:
- Review the pattern of false positives — time of day, equipment state, environmental conditions
- Check if environmental factors (temperature, humidity) changed recently
- Verify the correct asset type is configured

MAY:
- Compare alert patterns across sensors on similar equipment
- Check if facility operations changed (new equipment nearby, different shifts)

## Phase 2 — Remediate

MUST:
- Physically inspect equipment when alerts fire to confirm false positive
- Verify sensor is mounted correctly on the bearing housing
- Ensure learning period was completed during normal (not abnormal) operating conditions

SHOULD:
- Reset learning period if baseline was established during abnormal conditions
- Acknowledge false positives in the app to improve model feedback

MAY:
- Relocate sensor if environmental interference is identified
- Add vibration isolation if nearby equipment causes interference

## Common Issues

- symptoms: "Frequent anomaly alerts but equipment inspection shows no issues"
  diagnosis: "Sensor mounted on wrong location or learning period captured abnormal baseline."
  resolution: "Verify mounting position. If baseline was wrong, reset learning period during normal operations."

- symptoms: "Alerts correlate with specific times or events"
  diagnosis: "Normal operational variation (startup, shutdown, load changes) triggers alerts."
  resolution: "These may be expected variations. Document the pattern. Consider if the model needs retraining after a full operational cycle."

## Safety Ratings

```
safety_ratings:
  - "Equipment inspection: GREEN — physical observation only"
  - "Acknowledging alerts: GREEN — feedback to the system"
  - "Resetting learning period: YELLOW — 14 days without reliable detection"
  - "Sensor relocation: YELLOW — requires re-pairing and new learning period"
```

## Escalation Conditions

- False positive rate exceeds 50% causing alert fatigue
- Maintenance team stops responding to alerts due to false positive frequency
- False positives on safety-critical equipment undermine trust in the system
- Environmental interference cannot be resolved with sensor relocation

## Data Sensitivity

- Alert history: MEDIUM — reveals equipment monitoring patterns
- Equipment inspection results: LOW — maintenance observations
- Sensor placement details: LOW — physical location information
- Operational patterns: MEDIUM — reveals production schedules

## Prohibited Actions

- NEVER ignore all alerts because of false positive history — real anomalies can occur
- NEVER disable alerting to reduce false positives
- NEVER remove sensors from equipment to stop false alerts
- NEVER assume all alerts are false positives without physical inspection

## Phase 3 — Rollback

1. If learning period was reset, wait 14 days for new baseline
2. If sensor was relocated, verify new position provides accurate readings
3. If alert acknowledgments were made incorrectly, review and correct in the app
4. Document false positive patterns for model improvement
5. Restore any disabled notifications after root cause is addressed

## Output Format

```yaml
root_cause: "false_positive — <specific_cause>"
evidence:
  - type: alert_analysis
    content: "<alert pattern, inspection results, mounting details>"
severity: MEDIUM
mitigation:
  immediate: "Verify mounting and reset learning if needed"
  long_term: "Document operational patterns and optimize sensor placement"
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
