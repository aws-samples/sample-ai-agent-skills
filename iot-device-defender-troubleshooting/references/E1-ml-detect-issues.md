---
title: "E1 — ML Detect Issues"
description: "Diagnose ML Detect model training and anomaly detection issues"
status: active
severity: MEDIUM
triggers: ["ML Detect not working", "model training failed", "ML anomaly not detected", "ML model status error", "training period"]
owner: devops-agent
objective: "Resolve ML Detect issues including model training, data requirements, and anomaly detection accuracy"
context: "ML Detect uses machine learning to establish device behavior baselines and detect anomalies. Models require a 14-day training period with sufficient data. Models are per-behavior in a security profile."
---

## Phase 1 — Triage

MUST:
- Check security profile ML Detect configuration: `aws iot describe-security-profile --security-profile-name <name>`
- Check ML Detect model status: `aws iot get-behavior-model-training-summaries --security-profile-name <name>`
- Verify devices are reporting metrics during training period

SHOULD:
- Check model training status (IN_PROGRESS, ACTIVE, FAILED)
- Verify sufficient data points are being collected (minimum devices and data volume)
- Review active violations from ML Detect behaviors

MAY:
- Compare ML Detect violations with static threshold violations
- Check CloudWatch for ML model metrics

## Phase 2 — Remediate

MUST:
- Ensure devices report metrics consistently during the 14-day training period
- Fix data collection issues preventing model training
- Verify security profile has ML Detect behaviors configured (not just static thresholds)

SHOULD:
- Wait for training period to complete before expecting violations
- Ensure sufficient device diversity in the training data
- Monitor model accuracy after training completes

MAY:
- Adjust ML Detect confidence level for sensitivity tuning
- Combine ML Detect with static thresholds for defense in depth

## Safety Ratings

```
safety_ratings:
  - "iot describe-security-profile: GREEN — read-only profile check"
  - "iot get-behavior-model-training-summaries: GREEN — read-only model status"
  - "iot update-security-profile (ML behaviors): YELLOW — modifies detection configuration"
  - "ML model retraining: YELLOW — resets learned baselines"
```

## Escalation Conditions

- ML model training fails repeatedly with insufficient data
- ML Detect generates excessive false positives after training
- Model accuracy degrades over time indicating concept drift
- ML Detect misses known anomalous behavior

## Data Sensitivity

- `aws iot describe-security-profile`: MEDIUM — contains behavior and ML configuration
- `aws iot get-behavior-model-training-summaries`: LOW — returns model training status
- `ML model training data`: MEDIUM — derived from device behavior patterns
- `ML Detect violations`: MEDIUM — contain device identifiers and anomaly scores

## Prohibited Actions

- NEVER reset ML model training without understanding the impact on detection capability
- NEVER disable ML Detect to suppress false positives without investigating root cause
- NEVER assume ML Detect works during the training period — no violations are generated
- NEVER rely solely on ML Detect without static threshold fallbacks for critical metrics

## Phase 3 — Rollback

1. If ML Detect was misconfigured, revert security profile to previous behavior configuration
2. If model training was reset, wait 14 days for retraining to complete
3. If ML behaviors were removed, re-add them and wait for training
4. Verify static threshold behaviors are still active as fallback
5. Document the ML Detect issue and model training timeline

## Common Issues

- symptoms: "ML Detect not generating violations — model status IN_PROGRESS"
  diagnosis: "ML model is still in the 14-day training period."
  resolution: "Wait for training to complete. Ensure devices report metrics consistently."

## Output Format

```yaml
root_cause: "ml_detect — <specific_cause>"
evidence:
  - type: ml_model_status
    content: "<model training status and details>"
severity: MEDIUM
mitigation:
  immediate: "Fix ML Detect configuration"
  long_term: "Monitor model accuracy and retrain periodically"
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
