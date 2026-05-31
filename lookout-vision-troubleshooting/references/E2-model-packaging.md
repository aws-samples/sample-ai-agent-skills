---
title: "E2 — Model Packaging"
description: "Diagnose model packaging job failures for edge deployment"
status: active
severity: HIGH
triggers: ["packaging failed", "packaging job error", "cannot package model", "component creation failed"]
owner: devops-agent
objective: "Resolve model packaging issues for Greengrass component creation"
context: "Model packaging converts a trained model into a Greengrass V2 component for edge deployment. Packaging requires a trained model, proper IAM permissions, and an S3 output location."
---

## Phase 1 — Triage

MUST:
- List packaging jobs: `aws lookoutvision list-model-packaging-jobs --project-name <project>`
- Get packaging job details: `aws lookoutvision describe-model-packaging-job --project-name <project> --job-name <job-name>`
- Verify model is trained: `aws lookoutvision describe-model --project-name <project> --model-version <version>`

SHOULD:
- Check IAM role permissions for S3 output bucket
- Verify output S3 bucket exists and is accessible

MAY:
- Review CloudTrail for packaging job events
- Check Greengrass component registry for created components

## Phase 2 — Remediate

MUST:
- Ensure model is in TRAINED status (not TRAINING or TRAINING_FAILED)
- Grant s3:PutObject permission on the output bucket to the service role
- Verify the packaging job configuration specifies correct target platform

SHOULD:
- Ensure output S3 bucket is in the same region as the project
- Verify KMS key permissions if output bucket uses SSE-KMS

MAY:
- Test packaging with default configuration before customizing
- Check Greengrass component size limits for the target device

## Common Issues

- symptoms: "Packaging job fails with permission error"
  diagnosis: "Service role lacks s3:PutObject on the output bucket."
  resolution: "Add s3:PutObject permission for the output bucket. Verify KMS permissions if using SSE-KMS."

- symptoms: "Packaging job completes but Greengrass component not found"
  diagnosis: "Component was created in a different region or with unexpected name."
  resolution: "Check Greengrass components in the same region. Verify component name matches expected pattern."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision list-model-packaging-jobs: GREEN — read-only enumeration"
  - "lookoutvision describe-model-packaging-job: GREEN — read-only details"
  - "lookoutvision create-model-packaging-job: YELLOW — creates packaging job, incurs charges"
  - "s3api put-bucket-policy: YELLOW — modifies bucket access"
```

## Escalation Conditions

- Packaging job fails with internal errors after fixing permissions
- Target platform is not supported for the model type
- Output S3 bucket is in a different account requiring cross-account access
- Greengrass component size exceeds edge device storage capacity

## Data Sensitivity

- `aws lookoutvision list-model-packaging-jobs`: LOW — returns job metadata
- `aws lookoutvision describe-model-packaging-job`: MEDIUM — reveals packaging configuration
- Model packaging artifacts in S3: HIGH — contains trained model
- Greengrass component: HIGH — contains deployable model

## Prohibited Actions

- NEVER delete packaging job output without verifying no deployments reference it
- NEVER modify S3 bucket policies to grant public access to model artifacts
- NEVER share model packaging artifacts outside the organization
- NEVER package untested models for production edge deployment

## Phase 3 — Rollback

1. If packaging job produced incorrect component, delete the Greengrass component version
2. If S3 output was written to wrong location, move or delete with authorization
3. If IAM permissions were changed incorrectly, restore previous policy
4. Verify no edge deployments reference the incorrect component
5. Document the packaging issue for the ML operations team

## Output Format

```yaml
root_cause: "model_packaging — <specific_cause>"
evidence:
  - type: packaging_error
    content: "<job status, error details, S3/IAM configuration>"
severity: HIGH
mitigation:
  immediate: "Fix permissions and retry packaging"
  long_term: "Automate packaging pipeline with validation"
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
