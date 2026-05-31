---
title: "C2 — Query Result Location"
description: "Diagnose and resolve Athena query result location issues"
status: active
severity: HIGH
triggers:
  - "result location"
  - "output location"
  - "s3 output"
  - "cannot write results"
  - "InvalidRequestException"
owner: devops-agent
objective: "Identify and fix query result S3 location access issues"
context: "Every Athena query writes results to an S3 location. The executing principal must have write access. Location is set per workgroup or per query."
---

## Phase 1 — Triage

MUST:
- Check workgroup result location: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.ResultConfiguration.OutputLocation'`
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <results-bucket>`
- Check IAM permissions for result writing: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:PutObject s3:GetBucketLocation s3:AbortMultipartUpload --resource-arns arn:aws:s3:::<bucket>/*`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <results-bucket>`

SHOULD:
- Verify encryption configuration matches bucket policy requirements
- Check if bucket has Block Public Access settings that might interfere
- Verify the S3 path prefix exists or can be created

MAY:
- Check S3 bucket versioning and lifecycle rules
- Verify cross-account access if bucket is in a different account

## Phase 2 — Remediate

MUST:
- If no write permission: add s3:PutObject to IAM policy for the result bucket
- If bucket doesn't exist: create the bucket or update the result location
- If encryption mismatch: align workgroup encryption with bucket policy requirements
- If cross-account: add bucket policy allowing the Athena principal

SHOULD:
- Use a dedicated bucket for Athena results with lifecycle rules for cleanup
- Configure encryption at the workgroup level for consistent security

MAY:
- Set up S3 lifecycle rules to expire old query results
- Use separate result locations per workgroup for isolation

## Common Issues

- symptoms: "InvalidRequestException: Unable to verify/create output bucket"
  diagnosis: "S3 bucket does not exist or caller lacks s3:GetBucketLocation."
  resolution: "Create the bucket or add s3:GetBucketLocation permission."

- symptoms: "AccessDenied when writing query results"
  diagnosis: "IAM policy or bucket policy denies s3:PutObject."
  resolution: "Add s3:PutObject permission for the result S3 path."

## Output Format

```yaml
root_cause: "Query result location — <specific_issue>"
evidence:
  - type: result_config
    content: "<output location and encryption settings>"
  - type: s3_permissions
    content: "<IAM and bucket policy details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 permissions for the query result location"
  long_term: "Use dedicated result bucket with proper IAM policies and lifecycle rules"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add s3:PutObject to IAM policy | YELLOW | IAM permission change — review scope |
| Create result bucket | GREEN | New resource creation — non-destructive |
| Align workgroup encryption with bucket policy | YELLOW | Security configuration change |
| Add cross-account bucket policy | YELLOW | Cross-account access change — review carefully |
| Set up S3 lifecycle rules | GREEN | Cost management — non-destructive |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Cross-account S3 access changes require security review
- Encryption configuration changes affect data protection posture

## Data Sensitivity

- Query result locations contain actual query output — may include sensitive data
- S3 bucket policies control access to potentially sensitive query results
- Encryption settings protect query results at rest
- Cross-account access may expose results to additional principals

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER create unencrypted result buckets for workgroups processing sensitive data
- NEVER grant overly broad S3 bucket policies (e.g., Principal: "*")
- NEVER disable result encryption without security approval

## Phase 3 — Rollback

- If IAM permissions were changed: revert IAM policy to previous version
- If bucket policy was modified: restore original bucket policy
- If encryption settings were changed: revert workgroup encryption configuration
- If new bucket was created incorrectly: remove the bucket after verifying no data loss
- Document all permission and encryption changes for audit trail

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
  - "NEVER suggest disabling query result encryption"
  - "NEVER suggest overly broad S3 access for workgroups"
  - "NEVER suggest disabling workgroup enforcement"
