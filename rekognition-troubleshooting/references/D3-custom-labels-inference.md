---
title: "D3 — Custom Labels Inference Issues"
description: "Diagnose Custom Labels model inference problems"
status: active
severity: HIGH
triggers:
  - "DetectCustomLabels error"
  - "model not running"
  - "inference failed"
  - "custom labels not detecting"
owner: devops-agent
objective: "Identify and resolve Custom Labels inference issues"
context: "Custom Labels models must be explicitly started before inference. Models are billed while running. DetectCustomLabels requires a running model version."
---

## Phase 1 — Triage

MUST:
- Check model status: `aws rekognition describe-project-versions --project-arn <project-arn> --version-names <version>`
- Verify model is RUNNING (not STOPPED or STARTING)
- Test inference: `aws rekognition detect-custom-labels --project-version-arn <version-arn> --image '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}'`
- Check image format and size

SHOULD:
- Verify MinConfidence threshold setting
- Check inference unit count (affects throughput)
- Test with known positive images

MAY:
- Monitor inference latency and error rates
- Check for throttling on DetectCustomLabels

## Phase 2 — Remediate

MUST:
- Start model if stopped: `aws rekognition start-project-version --project-version-arn <arn> --min-inference-units 1`
- Wait for RUNNING status before inference
- Use supported image format

SHOULD:
- Set appropriate MinConfidence for use case
- Scale inference units for higher throughput

MAY:
- Implement model lifecycle management (start/stop scheduling)

## Common Issues

- symptoms: "DetectCustomLabels returns ResourceNotReadyException"
  diagnosis: "Model not started or still starting."
  resolution: "Start model. Wait for RUNNING status (several minutes)."

- symptoms: "Model running but returns no labels"
  diagnosis: "Image doesn't match training data or MinConfidence too high."
  resolution: "Lower MinConfidence. Verify image is similar to training data."

## Output Format

```yaml
root_cause: "custom_labels_inference — <specific_cause>"
evidence:
  - type: model_status
    content: "<model version status>"
  - type: inference_result
    content: "<DetectCustomLabels response>"
severity: HIGH
mitigation:
  immediate: "Start model or fix inference configuration"
  long_term: "Implement model lifecycle and monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Images and video may contain biometric data (faces, persons)
- Analysis results may contain personally identifiable information
- Custom Labels training data may contain proprietary content
- Classification: HIGHLY CONFIDENTIAL - biometric and PII data requires special handling

## Prohibited Actions

- NEVER delete face collections containing production data
- NEVER stop running Custom Labels models without confirming downstream impact
- NEVER modify content moderation thresholds in production without approval
- NEVER process PII-containing images without confirming data handling compliance

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
