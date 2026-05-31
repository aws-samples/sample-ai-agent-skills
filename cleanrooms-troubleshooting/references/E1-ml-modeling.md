---
title: "E1 — ML Modeling Issues"
description: "Diagnose Clean Rooms ML modeling problems including training dataset and model configuration"
status: active
severity: MEDIUM
triggers:
  - "ml modeling"
  - "training dataset"
  - "clean rooms ml"
  - "model training"
  - "ml configuration"
owner: devops-agent
objective: "Resolve Clean Rooms ML modeling issues and enable ML collaboration"
context: "Clean Rooms ML enables collaborative ML modeling without sharing raw data. It uses the cleanrooms-ml API. Common issues include training dataset configuration, model training failures, IAM permissions, and data format errors."
---

## Phase 1 — Triage

MUST:
- List training datasets: `aws cleanrooms-ml list-training-datasets`
- Check training dataset status and configuration
- Verify IAM permissions include `cleanrooms-ml:*` actions
- Check if the collaboration has ML capabilities enabled

SHOULD:
- Verify training data format matches requirements
- Check S3 data access for training datasets
- Review model training logs for errors

MAY:
- Check CloudTrail for cleanrooms-ml events
- Verify service quotas for ML resources

## Phase 2 — Remediate

MUST:
- Fix training dataset configuration if format errors
- Grant required cleanrooms-ml IAM permissions
- Ensure S3 data is accessible to the Clean Rooms ML service role

SHOULD:
- Validate training data format before creating datasets
- Monitor model training progress

## Output Format

```yaml
root_cause: "ml_modeling — <specific_cause>"
evidence:
  - type: training_dataset
    content: "<training dataset details>"
  - type: model_status
    content: "<model training status>"
severity: MEDIUM
mitigation:
  immediate: "Fix dataset configuration or permissions"
  long_term: "Automate ML pipeline with validation steps"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix training dataset configuration | YELLOW |
| Grant cleanrooms-ml IAM permissions | YELLOW |
| Ensure S3 data is accessible to service role | YELLOW |
| Validate training data format | GREEN |
| Monitor model training progress | GREEN |

## Escalation Conditions

- Fix requires modifying production ML model configurations
- Fix involves changing training data access or IAM permissions

## Data Sensitivity

- **HIGH**: ML training datasets contain sensitive collaborative data
- Model configurations and training parameters may reveal business intelligence strategies

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If training dataset was misconfigured, delete and recreate with correct settings
2. If IAM permissions were broadened, revert to previous policy
3. If S3 access was modified, restore previous bucket policy
4. Verify rollback by confirming training dataset and model configurations match expected state

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
