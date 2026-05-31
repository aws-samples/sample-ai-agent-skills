---
title: "D1 — Custom Model Training Failures"
description: "Diagnose Bedrock custom model fine-tuning failures"
status: active
severity: HIGH
triggers:
  - "training failed"
  - "fine-tuning error"
  - "CreateModelCustomizationJob failed"
  - "custom model training"
owner: devops-agent
objective: "Identify and resolve custom model training failures"
context: "Custom model training (fine-tuning) requires JSONL training data in S3, proper IAM role, and supported base model. Training can take hours. Failures stem from data format issues, insufficient data, IAM permissions, or S3 access problems."
---

## Phase 1 — Triage

MUST:
- Check training job status: `aws bedrock get-model-customization-job --job-identifier <job-arn>`
- List training jobs: `aws bedrock list-model-customization-jobs`
- Verify training data format (JSONL with model-specific schema)
- Check training role permissions

SHOULD:
- Verify S3 training data accessibility
- Check training data volume (minimum requirements vary by model)
- Review training hyperparameters
- Check CloudTrail for training job events

MAY:
- Validate training data format independently
- Check for training job quotas

## Phase 2 — Remediate

MUST:
- Fix training data format (JSONL, model-specific schema)
- Ensure training role has S3 and Bedrock permissions
- Verify base model supports fine-tuning

SHOULD:
- Validate training data before submitting job
- Start with minimum data requirements and scale up
- Monitor training job progress

MAY:
- Create training data validation scripts
- Implement training pipeline automation

## Common Issues

- symptoms: "Training job fails with data format error"
  diagnosis: "JSONL format incorrect or schema doesn't match model requirements."
  resolution: "Verify JSONL format. Check model-specific training data schema."

- symptoms: "Training job fails with access denied"
  diagnosis: "Training role lacks S3 or Bedrock permissions."
  resolution: "Add s3:GetObject, s3:PutObject, and bedrock:* to the training role."

## Output Format

```yaml
root_cause: "custom_model_training — <specific_cause>"
evidence:
  - type: job_status
    content: "<training job status and error>"
  - type: data_format
    content: "<training data format analysis>"
severity: HIGH
mitigation:
  immediate: "Fix training data or permissions and retry"
  long_term: "Create training data validation pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix training data format (JSONL, model-specific schema) | GREEN |
| Ensure training role has S3 and Bedrock permissions | YELLOW |
| Verify base model supports fine-tuning | GREEN |
| Validate training data before submitting job | GREEN |
| Start with minimum data requirements and scale up | GREEN |
| Monitor training job progress | GREEN |
| Create training data validation scripts | GREEN |
| Implement training pipeline automation | YELLOW |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Training data in S3 may contain proprietary, sensitive, or PII data requiring strict access controls

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If a training job was started with incorrect data, stop it: `aws bedrock stop-model-customization-job --job-identifier <job-arn>`
2. If a custom model was created from bad training data, delete it: `aws bedrock delete-custom-model --model-identifier <model-id>`
3. If IAM roles were broadened, revert the training role policy to the previous version
4. If training data was uploaded to S3, remove incorrect data and restore previous dataset
5. Verify rollback by confirming `aws bedrock list-custom-models` shows expected state

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
