---
title: "D2 — Retraining Policy"
description: "Diagnose Amazon Lookout for Equipment retraining scheduler and policy issues"
status: active
severity: MEDIUM
triggers: ["retraining failed", "retraining scheduler error", "model not retraining", "retraining policy issues"]
owner: devops-agent
objective: "Resolve retraining scheduler issues to maintain model accuracy over time"
context: "Retraining schedulers automatically retrain models on new data to prevent model drift. They require access to original training data plus new data in S3."
---

## Phase 1 — Triage

MUST:
- Check retraining scheduler: `aws lookoutequipment describe-retraining-scheduler --model-name <name>`
- Verify the model is in ACTIVE status: `aws lookoutequipment describe-model --model-name <name>`
- Check that original training data is still accessible in S3
- Verify new data has been ingested since the last training

SHOULD:
- Check CloudTrail for retraining-related events
- Verify the retraining lookback window covers sufficient data
- Check if the retraining schedule frequency is appropriate for data volume

MAY:
- Compare model metrics before and after retraining
- Review the retraining history for patterns of failure

## Phase 2 — Remediate

MUST:
- Ensure original training data remains in S3 at the original path
- Verify the IAM role has access to both original and new data
- Fix the retraining lookback window to include sufficient new data
- Ensure the model is ACTIVE before enabling retraining

SHOULD:
- Set retraining frequency based on data accumulation rate (weekly or monthly)
- Configure model quality thresholds to auto-promote or reject retrained models
- Monitor retraining job duration trends for early warning of issues

MAY:
- Implement A/B testing between original and retrained models
- Set up notifications for retraining completion and quality metrics

## Common Issues

- symptoms: "Retraining scheduler shows FAILED status"
  diagnosis: "Original training data was moved or deleted from S3, or new data is insufficient."
  resolution: "Restore original training data to the original S3 path. Ensure new data has been ingested into the dataset."

- symptoms: "Retrained model has worse performance than the original"
  diagnosis: "New data may contain anomalous periods that were not labeled, or equipment behavior has changed significantly."
  resolution: "Add labels for anomalous periods in new data. Review if equipment modifications require a fresh model rather than retraining."

- symptoms: "Retraining scheduler is not triggering at expected times"
  diagnosis: "The retraining schedule may be misconfigured or the scheduler may be in STOPPED status."
  resolution: "Verify the scheduler is in RUNNING status. Check the schedule frequency and next scheduled run time."

## Safety Ratings

```
safety_ratings:
  - "describe-retraining-scheduler: GREEN — read-only status check"
  - "CreateRetrainingScheduler: YELLOW — sets up automated retraining with compute costs"
  - "start-retraining-scheduler: YELLOW — initiates automated model updates"
  - "stop-retraining-scheduler: YELLOW — stops automated retraining, may cause model drift"
  - "Model promotion after retraining: RED — replaces production model"
```

## Escalation Conditions

- Retraining consistently produces worse models than the original
- Original training data was permanently lost and retraining cannot proceed
- Equipment behavior has fundamentally changed requiring a new model architecture
- Retraining costs are unexpectedly high due to data volume growth

## Data Sensitivity

- `aws lookoutequipment describe-retraining-scheduler`: LOW — returns schedule configuration
- Retraining metrics comparison: LOW — statistical measures
- Training data paths: LOW — S3 paths only
- Model version history: MEDIUM — reveals model evolution and equipment behavior changes

## Prohibited Actions

- NEVER auto-promote a retrained model to production without reviewing quality metrics
- NEVER delete original training data while a retraining scheduler is active
- NEVER disable retraining for production models without a documented justification
- NEVER modify the retraining lookback window without understanding the data accumulation rate

## Phase 3 — Rollback

1. If retraining produced a bad model, revert to the previous model version
2. Stop the retraining scheduler if it is producing consistently poor results: `aws lookoutequipment stop-retraining-scheduler --model-name <name>`
3. If original data was lost, restore from S3 versioning or backups before re-enabling retraining
4. Document the retraining failure and model quality metrics for the ML team
5. Update inference schedulers to use the known-good model version

## Output Format

```yaml
root_cause: "retraining_policy — <specific_cause>"
evidence:
  - type: retraining_error
    content: "<error details from describe-retraining-scheduler>"
severity: MEDIUM
mitigation:
  immediate: "Fix data access or configuration issues"
  long_term: "Implement model quality gates for automated retraining promotion"
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
