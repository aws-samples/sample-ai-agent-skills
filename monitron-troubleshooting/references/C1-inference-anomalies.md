---
title: "C1 — Inference Anomalies"
description: "Diagnose ML inference issues, missed anomalies, and detection accuracy problems"
status: active
severity: HIGH
triggers: ["missed anomaly", "no alert", "inference wrong", "detection not working", "anomaly not detected"]
owner: devops-agent
objective: "Resolve inference accuracy issues and ensure reliable anomaly detection"
context: "Monitron uses ML to detect vibration and temperature anomalies. The model requires a 14-day learning period and depends on correct sensor mounting and consistent baseline conditions."
---

## Phase 1 — Triage

MUST:
- Verify sensor has completed the 14-day learning period
- Check sensor data continuity — gaps affect model accuracy
- Confirm sensor is properly mounted on the bearing housing per guidelines

SHOULD:
- Review recent inference results in the Monitron mobile app or console
- Check if equipment operating conditions changed (speed, load, environment)
- Verify sensor orientation matches the configured mounting position

MAY:
- Compare inference results across sensors on similar equipment
- Check CloudWatch metrics for inference event patterns

## Phase 2 — Remediate

MUST:
- Allow full 14-day learning period before expecting reliable detection
- Verify sensor mounting position and orientation match configuration
- Ensure continuous data collection (no gaps) for accurate baseline

SHOULD:
- Recalibrate by resetting the learning period if equipment operating conditions changed significantly
- Verify the correct asset type is configured for the monitored equipment

MAY:
- Add additional sensors for critical equipment to improve coverage
- Adjust alert thresholds if available through the console

## Common Issues

- symptoms: "No anomaly alerts despite known equipment issue"
  diagnosis: "Sensor still in 14-day learning period or sensor mounted incorrectly."
  resolution: "Verify learning period is complete. Check sensor mounting position. If mounting was wrong, remount and restart learning."

- symptoms: "Inference results show normal but equipment is clearly degraded"
  diagnosis: "Equipment degradation occurred gradually during learning period — model learned degraded state as baseline."
  resolution: "Reset the sensor learning period after equipment is repaired to establish correct baseline."

## Safety Ratings

```
safety_ratings:
  - "Reviewing inference results: GREEN — read-only observation"
  - "Resetting learning period: YELLOW — 14 days without reliable detection"
  - "Sensor remounting: YELLOW — requires physical access, restarts learning"
  - "Adding sensors: YELLOW — new sensors need 14-day learning period"
```

## Escalation Conditions

- Anomaly detection consistently misses known equipment issues after learning period
- Safety-critical equipment has unreliable anomaly detection
- Multiple sensors on different equipment show similar detection failures
- Equipment failure occurred without any Monitron alert

## Data Sensitivity

- Inference results: MEDIUM — reveals equipment health status
- Vibration and temperature data: MEDIUM — reveals operational patterns
- Equipment asset information: LOW — asset identifiers and types
- Maintenance records: MEDIUM — reveals equipment history

## Prohibited Actions

- NEVER rely solely on Monitron for safety-critical equipment monitoring
- NEVER dismiss anomaly alerts without physical equipment inspection
- NEVER reset learning period on safety-critical equipment without backup monitoring
- NEVER modify inference thresholds without understanding the impact on detection accuracy

## Phase 3 — Rollback

1. If learning period was reset unnecessarily, it cannot be undone — wait 14 days
2. If sensor was remounted, verify new position is correct before waiting for learning
3. If alert thresholds were changed, restore previous values
4. Document the inference issue for the maintenance team
5. Implement backup monitoring for critical equipment during learning periods

## Output Format

```yaml
root_cause: "inference_anomaly — <specific_cause>"
evidence:
  - type: inference_issue
    content: "<sensor status, learning period, mounting details>"
severity: HIGH
mitigation:
  immediate: "Verify mounting and learning period"
  long_term: "Establish sensor installation standards and verification checklist"
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
