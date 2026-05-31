---
title: "A1 — Pipeline Creation Failures"
description: "Diagnose Elastic Transcoder pipeline creation failures"
status: active
severity: HIGH
triggers: ["cannot create pipeline", "pipeline creation failed", "CreatePipeline error"]
owner: devops-agent
objective: "Resolve pipeline creation issues including IAM role, S3 bucket, and region problems"
context: "Pipelines require a valid IAM role, input/output S3 buckets in the same region, and optional SNS topics."
---

## Phase 1 — Triage

MUST:
- List existing pipelines: `aws elastictranscoder list-pipelines`
- Verify IAM role exists and is assumable: `aws iam get-role --role-name <role-name>`
- Verify input bucket exists in the same region: `aws s3api get-bucket-location --bucket <bucket-name>`
- Verify output bucket exists in the same region: `aws s3api get-bucket-location --bucket <bucket-name>`

SHOULD:
- Check IAM role trust policy allows elastictranscoder.amazonaws.com to assume it
- Verify role has s3:GetObject on input bucket and s3:PutObject on output bucket
- Check if pipeline limit has been reached in the region

MAY:
- Review CloudTrail for CreatePipeline error details: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePipeline --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policy to include elastictranscoder.amazonaws.com as trusted service
- Ensure S3 buckets are in the same region as the pipeline
- Attach required S3 permissions to the IAM role

SHOULD:
- Configure SNS topics for job notifications (Completed, Warning, Error)
- Use separate output and thumbnail buckets for organization

MAY:
- Request pipeline limit increase via AWS Support if at capacity

## Safety Ratings

```
safety_ratings:
  - "list-pipelines: GREEN — read-only enumeration of existing pipelines"
  - "iam get-role: GREEN — read-only role inspection"
  - "s3api get-bucket-location: GREEN — read-only bucket metadata"
  - "create-pipeline: YELLOW — creates a new pipeline resource"
```

## Escalation Conditions

- Pipeline creation fails with internal service errors after all configuration is verified correct
- IAM role cannot be created or modified due to organizational SCPs
- S3 bucket access is blocked by bucket policy that the operator cannot modify
- Pipeline limit reached and AWS Support has not responded to increase request

## Data Sensitivity

- `aws elastictranscoder list-pipelines`: LOW — returns pipeline metadata only
- `aws iam get-role`: LOW — returns role configuration and trust policy
- `aws s3api get-bucket-location`: LOW — returns bucket region only
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions

## Prohibited Actions

- NEVER delete existing pipelines to free up pipeline limit without explicit approval
- NEVER modify IAM roles used by other services without verifying impact
- NEVER create pipelines with overly permissive IAM roles (s3:* on *)
- NEVER use the root account to create pipelines

## Phase 3 — Rollback

1. If a pipeline was created with incorrect configuration, delete it: `aws elastictranscoder delete-pipeline --id <pipeline-id>` (only after confirming no active jobs)
2. If an IAM role was created incorrectly, delete it: `aws iam delete-role --role-name <role-name>` (after detaching all policies)
3. If S3 buckets were created for the pipeline in error, verify they are empty before deletion
4. Revert any IAM policy changes that were made during troubleshooting
5. Document the rollback action and verify no jobs were affected

## Common Issues

- symptoms: "CreatePipeline returns ValidationException about role"
  diagnosis: "IAM role trust policy does not include elastictranscoder.amazonaws.com."
  resolution: "Update role trust policy: add elastictranscoder.amazonaws.com as a trusted service principal."

- symptoms: "CreatePipeline returns ValidationException about bucket"
  diagnosis: "S3 bucket is in a different region than the pipeline."
  resolution: "Create or use an S3 bucket in the same region as the pipeline."

## Output Format

```yaml
root_cause: "pipeline_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix pipeline configuration"
  long_term: "Use IaC templates for consistent pipeline creation"
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
