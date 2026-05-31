---
title: "D2 — Output Configuration"
description: "Diagnose audience generation output configuration and delivery issues"
status: active
severity: HIGH
triggers: ["output missing", "output configuration error", "S3 output failed", "results not delivered"]
owner: devops-agent
objective: "Resolve output configuration issues and ensure audience data delivery"
context: "Output configuration defines where audience generation results are written. Misconfigured output causes jobs to fail at the write stage after consuming privacy budget."
---

## Phase 1 — Triage

MUST:
- Check configured audience model output settings: `aws cleanroomsml get-configured-audience-model --configured-audience-model-arn <arn>`
- Verify output S3 bucket exists: `aws s3api head-bucket --bucket <output-bucket>`
- Check audience generation job status and error: `aws cleanroomsml get-audience-generation-job --audience-generation-job-arn <arn>`

SHOULD:
- Verify S3 bucket policy allows cleanroomsml.amazonaws.com to write: `aws s3api get-bucket-policy --bucket <output-bucket>`
- Check KMS key policy for encrypt permissions

MAY:
- List objects in the output prefix to see if partial output exists: `aws s3 ls s3://<output-bucket>/<prefix>/`
- Review CloudTrail for S3 PutObject failures

## Phase 2 — Remediate

MUST:
- Grant s3:PutObject permission to the Clean Rooms ML service role on the output bucket
- Ensure KMS key policy allows kms:GenerateDataKey and kms:Encrypt for the service role
- Verify output bucket is in the same region as the collaboration

SHOULD:
- Add s3:PutObjectAcl if bucket requires ACL-based access control
- Enable S3 server access logging on the output bucket for debugging

MAY:
- Create a dedicated output bucket with pre-configured permissions
- Use S3 bucket notifications to alert when output is delivered

## Common Issues

- symptoms: "Audience generation job fails at output stage"
  diagnosis: "Service role lacks s3:PutObject on the output bucket."
  resolution: "Add s3:PutObject and s3:PutObjectAcl to the service role policy for the output bucket ARN."

- symptoms: "Output bucket exists but no files appear after successful job"
  diagnosis: "KMS encryption failure — service role cannot encrypt output."
  resolution: "Update KMS key policy to grant kms:GenerateDataKey and kms:Encrypt to the service role."

## Safety Ratings

```
safety_ratings:
  - "s3api head-bucket: GREEN — read-only bucket existence check"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "s3api put-bucket-policy: YELLOW — modifies bucket access policy"
  - "kms put-key-policy: RED — modifies KMS key access, affects encryption"
```

## Escalation Conditions

- Output bucket is owned by another account and policy changes require cross-account coordination
- KMS key is managed by a security team and policy changes require approval
- Job consumed privacy budget but failed at output — budget is not refunded
- Output data governance requirements are unclear

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals access policy details
- `aws s3 ls s3://<output-bucket>/`: MEDIUM — reveals output file names
- `aws kms describe-key`: LOW — returns key metadata
- Audience generation output: HIGH — contains derived audience segments

## Prohibited Actions

- NEVER modify S3 bucket policies to grant public access
- NEVER change KMS key policies without approval from the key administrator
- NEVER redirect output to a bucket outside the collaboration agreement
- NEVER access output data without authorization from all collaboration members

## Phase 3 — Rollback

1. If bucket policy was changed incorrectly, restore previous policy: `aws s3api put-bucket-policy --bucket <bucket> --policy file://previous-policy.json`
2. If KMS key policy was modified, restore from backup: `aws kms put-key-policy --key-id <id> --policy-name default --policy file://previous-key-policy.json`
3. If output was written to wrong location, coordinate with data owner to move or delete
4. Verify no unauthorized access occurred during misconfiguration
5. Document the output configuration fix for the collaboration team

## Output Format

```yaml
root_cause: "output_configuration — <specific_cause>"
evidence:
  - type: output_error
    content: "<error details from job or S3/KMS inspection>"
severity: HIGH
mitigation:
  immediate: "Fix output bucket permissions and retry"
  long_term: "Use IaC to manage output bucket configuration"
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
