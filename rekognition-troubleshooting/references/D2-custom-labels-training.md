---
title: "D2 — Custom Labels Training Failures"
description: "Diagnose Custom Labels model training issues"
status: active
severity: HIGH
triggers:
  - "training failed"
  - "model training error"
  - "CreateProjectVersion failed"
  - "training taking too long"
owner: devops-agent
objective: "Identify and resolve Custom Labels training failures"
context: "Training creates a model version. Takes 30 min to 24 hours. Requires sufficient labeled data, S3 output location, and proper IAM permissions. Training costs apply."
---

## Phase 1 — Triage

MUST:
- Check model version status: `aws rekognition describe-project-versions --project-arn <project-arn>`
- Review training status and failure message
- Verify training dataset has sufficient labeled images
- Check S3 output bucket permissions

SHOULD:
- Verify training/test split is appropriate
- Check for class imbalance in training data
- Review training output metrics (if available)

MAY:
- Check CloudTrail for CreateProjectVersion events
- Review training cost estimates

## Phase 2 — Remediate

MUST:
- Provide sufficient training data (10+ images per label)
- Grant S3 write access for training output
- Fix any data format issues

SHOULD:
- Balance classes in training data
- Use test dataset for evaluation
- Review model performance metrics after training

MAY:
- Implement training pipeline with validation

## Common Issues

- symptoms: "Training fails with TRAINING_FAILED status"
  diagnosis: "Insufficient data, S3 permission error, or data format issue."
  resolution: "Check status message. Fix data or permissions. Retry training."

- symptoms: "Model has poor accuracy after training"
  diagnosis: "Insufficient or low-quality training data."
  resolution: "Add more diverse, high-quality labeled images. Retrain."

## Output Format

```yaml
root_cause: "custom_labels_training — <specific_cause>"
evidence:
  - type: model_version
    content: "<model version status>"
  - type: training_metrics
    content: "<training performance metrics>"
severity: HIGH
mitigation:
  immediate: "Fix training data or permissions"
  long_term: "Implement training data quality pipeline"
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
