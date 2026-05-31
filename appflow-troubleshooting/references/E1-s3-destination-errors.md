---
title: "E1 — S3 Destination Errors"
description: "Diagnose Amazon AppFlow S3 destination write failures"
status: active
severity: HIGH
triggers: ["S3 destination error", "cannot write to S3", "S3 permission denied", "S3 bucket error"]
owner: devops-agent
objective: "Resolve S3 destination issues including permissions, bucket policies, and output format errors"
context: "AppFlow writes to S3 using structured prefixes. Requires s3:PutObject, s3:GetBucketAcl permissions. Output format can be JSON, CSV, or Parquet."
---

## Phase 1 — Triage

MUST:
- Check flow destination config: `aws appflow describe-flow --flow-name <flow-name>`
- Verify S3 bucket exists and is in the same region: `aws s3api head-bucket --bucket <bucket-name>`
- Check bucket policy allows AppFlow writes: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Verify IAM role has s3:PutObject permission on the destination prefix
- Check if bucket has encryption requirements: `aws s3api get-bucket-encryption --bucket <bucket-name>`

MAY:
- Review S3 access logs for denied requests
- Check if S3 Block Public Access settings interfere: `aws s3api get-public-access-block --bucket <bucket-name>`

## Phase 2 — Remediate

MUST:
- Fix bucket policy to allow appflow.amazonaws.com PutObject access
- Ensure IAM role has s3:PutObject, s3:GetBucketAcl on the target bucket
- If KMS-encrypted bucket, grant appflow.amazonaws.com kms:GenerateDataKey

SHOULD:
- Verify output prefix path is correct
- Test with a small on-demand execution

MAY:
- Configure S3 event notifications for downstream processing

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow details"
  - "s3api head-bucket: GREEN — read-only bucket check"
  - "s3api get-bucket-policy: GREEN — read-only policy review"
  - "s3api put-bucket-policy: RED — modifies bucket access policy"
```

## Escalation Conditions

- Bucket policy changes require approval from the security team
- Cross-account S3 destination requires both account policies
- KMS key policy changes affect other services using the same key
- S3 destination errors cause data loss in production pipelines

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access configuration
- `aws s3api get-bucket-encryption`: LOW — reveals encryption configuration
- `aws appflow describe-flow`: MEDIUM — reveals destination configuration

## Prohibited Actions

- NEVER grant s3:* permissions as a troubleshooting shortcut
- NEVER disable bucket encryption to fix write errors
- NEVER modify bucket policies without understanding the impact on other services
- NEVER delete output files from previous successful executions

## Phase 3 — Rollback

1. If bucket policy was changed, revert to previous policy
2. If KMS key policy was modified, restore original key policy
3. Remove any test data written during troubleshooting
4. Verify other services using the same bucket are not affected
5. Document the correct bucket policy for AppFlow access

## Common Issues

- symptoms: "AccessDenied when writing to S3 destination"
  diagnosis: "Bucket policy does not allow appflow.amazonaws.com to PutObject."
  resolution: "Add bucket policy statement allowing appflow.amazonaws.com s3:PutObject on the bucket and prefix."

## Output Format

```yaml
root_cause: "s3_destination — <specific_cause>"
evidence:
  - type: s3_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 permissions"
  long_term: "Use IaC for bucket policies"
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
