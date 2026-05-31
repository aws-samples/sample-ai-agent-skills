---
title: "A1 — Training Model Failures"
description: "Diagnose training model creation and execution failures in Clean Rooms ML"
status: active
severity: HIGH
triggers: ["training failed", "model training error", "CREATE_FAILED", "training dataset error"]
owner: devops-agent
objective: "Resolve training model failures and get models to ACTIVE status"
context: "Training models require valid training datasets with correct schemas, proper IAM roles, S3 access, and KMS permissions. Training is asynchronous and can take hours."
---

## Phase 1 — Triage

MUST:
- List training datasets and check status: `aws cleanroomsml list-training-datasets --query 'trainingDatasets[*].{Name:name,Arn:trainingDatasetArn,Status:status}'`
- Get details of the failing training dataset: `aws cleanroomsml get-training-dataset --training-dataset-arn <arn>`
- Check CloudTrail for error events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cleanroomsml.amazonaws.com --max-results 20`

SHOULD:
- Verify the S3 source data exists and is accessible: `aws s3 ls s3://<bucket>/<prefix>/`
- Check IAM role trust policy includes cleanroomsml.amazonaws.com service principal

MAY:
- Review CloudWatch metrics for the training job
- Validate CSV schema against the training dataset definition

## Phase 2 — Remediate

MUST:
- Fix schema mismatches between CSV data and training dataset definition
- Ensure IAM role has s3:GetObject, s3:GetBucketLocation, and kms:Decrypt permissions
- Verify S3 bucket is in the same region as the collaboration

SHOULD:
- Add S3 bucket policy allowing cleanroomsml.amazonaws.com access
- Enable CloudTrail logging for cleanroomsml events for future debugging

MAY:
- Create a test training dataset with a small data sample to validate configuration before full training

## Common Issues

- symptoms: "Training dataset status is CREATE_FAILED with no clear error"
  diagnosis: "Schema mismatch between declared columns and actual CSV data."
  resolution: "Compare training dataset column definitions with CSV headers and data types. Recreate training dataset with corrected schema."

- symptoms: "AccessDeniedException during training"
  diagnosis: "IAM role missing required permissions for S3 or KMS."
  resolution: "Add s3:GetObject, s3:ListBucket, kms:Decrypt, kms:DescribeKey to the service role. Verify trust policy."

## Safety Ratings

```
safety_ratings:
  - "list-training-datasets: GREEN — read-only enumeration"
  - "get-training-dataset: GREEN — read-only detail retrieval"
  - "create-training-dataset: YELLOW — creates a new resource, consumes time"
  - "delete-training-dataset: RED — permanently removes training dataset and associated metadata"
```

## Escalation Conditions

- Training fails repeatedly with internal service errors after fixing IAM and schema
- Training dataset creation succeeds but model training hangs in CREATING state for over 6 hours
- Multiple training datasets fail across different collaborations suggesting account-level issue
- KMS key policy changes require key administrator approval

## Data Sensitivity

- `aws cleanroomsml list-training-datasets`: LOW — returns metadata only
- `aws cleanroomsml get-training-dataset`: MEDIUM — reveals schema and S3 locations
- `aws s3 ls s3://<bucket>/<prefix>/`: MEDIUM — reveals data file names and sizes
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions

## Prohibited Actions

- NEVER download or inspect the actual training data content — it belongs to collaboration members
- NEVER delete a training dataset that is referenced by an active audience model
- NEVER modify S3 bucket policies without approval from the data owner
- NEVER share training dataset ARNs or S3 paths outside the collaboration context

## Phase 3 — Rollback

1. If a training dataset was created with wrong schema, delete it: `aws cleanroomsml delete-training-dataset --training-dataset-arn <arn>` (only if no active models reference it)
2. If IAM role was modified incorrectly, restore from the previous policy version
3. If S3 bucket policy was changed, revert to the previous policy: `aws s3api put-bucket-policy --bucket <bucket> --policy file://previous-policy.json`
4. Verify no audience models were affected by checking their status
5. Document the failure and root cause for the collaboration team

## Output Format

```yaml
root_cause: "training_model_failure — <specific_cause>"
evidence:
  - type: training_error
    content: "<error details from get-training-dataset or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix schema/permissions and retry training"
  long_term: "Implement schema validation before training dataset creation"
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
