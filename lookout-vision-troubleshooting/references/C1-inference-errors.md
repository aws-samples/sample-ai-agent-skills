---
title: "C1 — Inference Errors"
description: "Diagnose inference failures, incorrect predictions, and model hosting issues"
status: active
severity: HIGH
triggers: ["inference error", "detect-anomalies failed", "model not responding", "wrong prediction", "model not started"]
owner: devops-agent
objective: "Resolve inference errors and ensure reliable anomaly detection"
context: "Inference requires a model in HOSTED status. Images must be sent via detect-anomalies API. The model must be explicitly started before inference and stopped when not needed."
---

## Phase 1 — Triage

MUST:
- Check model status: `aws lookoutvision describe-model --project-name <project> --model-version <version>` (must be HOSTED)
- Verify the model is started: status should be HOSTED, not TRAINED
- Test inference with a known image: `aws lookoutvision detect-anomalies --project-name <project> --model-version <version> --content-type image/jpeg --body fileb://test-image.jpg`

SHOULD:
- Check if the inference image matches training image characteristics
- Verify IAM permissions for lookoutvision:DetectAnomalies
- Review inference response for confidence scores

MAY:
- Compare inference results across multiple test images
- Check CloudWatch metrics for inference latency and error rates

## Phase 2 — Remediate

MUST:
- Start the model if not in HOSTED status: `aws lookoutvision start-model --project-name <project> --model-version <version> --min-inference-units 1`
- Ensure inference images are PNG or JPEG format
- Verify IAM role has lookoutvision:DetectAnomalies permission

SHOULD:
- Match inference image characteristics to training images (resolution, lighting, angle)
- Set appropriate confidence threshold for the use case

MAY:
- Implement retry logic for transient inference errors
- Set up CloudWatch alarms for inference error rates

## Common Issues

- symptoms: "detect-anomalies returns ResourceNotFoundException"
  diagnosis: "Model is not started — status is TRAINED, not HOSTED."
  resolution: "Start the model: aws lookoutvision start-model. Wait for status to transition to HOSTED (several minutes)."

- symptoms: "Inference returns incorrect predictions consistently"
  diagnosis: "Inference images differ significantly from training images."
  resolution: "Ensure inference images match training conditions (lighting, angle, resolution). Retrain with more representative images if needed."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision describe-model: GREEN — read-only model details"
  - "lookoutvision detect-anomalies: GREEN — read-only inference, no state change"
  - "lookoutvision start-model: YELLOW — starts model hosting, incurs hourly charges"
  - "lookoutvision stop-model: YELLOW — stops inference capability"
```

## Escalation Conditions

- Model fails to transition to HOSTED status after start-model
- Inference consistently returns incorrect results despite matching image conditions
- Inference latency exceeds acceptable thresholds for the production line
- Model hosting costs exceed budget

## Data Sensitivity

- `aws lookoutvision describe-model`: MEDIUM — reveals model configuration
- `aws lookoutvision detect-anomalies`: HIGH — processes potentially proprietary images
- Inference results: MEDIUM — reveals product quality information
- Confidence scores: LOW — numerical predictions

## Prohibited Actions

- NEVER leave models in HOSTED status when not actively serving inference — costs accrue hourly
- NEVER use inference results as the sole quality control mechanism without human oversight
- NEVER send images containing PII or sensitive data for inference
- NEVER share inference results outside the authorized quality control team

## Phase 3 — Rollback

1. If model was started unnecessarily, stop it: `aws lookoutvision stop-model --project-name <project> --model-version <version>`
2. If wrong model version was started, stop it and start the correct version
3. If confidence threshold was changed, restore previous value in the application
4. Verify inference pipeline is using the correct model version
5. Document the inference issue for the ML operations team

## Output Format

```yaml
root_cause: "inference_error — <specific_cause>"
evidence:
  - type: inference_error
    content: "<model status, error message, confidence scores>"
severity: HIGH
mitigation:
  immediate: "Start model and fix image pipeline"
  long_term: "Implement inference monitoring and auto-scaling"
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
