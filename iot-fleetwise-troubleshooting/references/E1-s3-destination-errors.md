---
title: "E1 — S3 Destination Errors"
description: "Diagnose S3 destination configuration, permission, and data delivery failures"
status: active
severity: MEDIUM
triggers: ["S3 destination error", "data not in S3", "S3 permission denied", "campaign S3 delivery failed"]
owner: devops-agent
objective: "Resolve S3 destination issues for FleetWise campaign data delivery"
context: "S3 destinations store collected vehicle data in batched files. Data is partitioned by vehicle name and time. Requires IAM role with S3 write permissions and correct bucket configuration."
---

## Phase 1 — Triage

MUST:
- Check campaign destination configuration: `aws iotfleetwise get-campaign --name <name> --query 'dataDestinationConfigs'`
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <bucket-name>`
- Check IAM role for S3 write permissions: `aws iam get-role-policy --role-name <role> --policy-name <policy>`

SHOULD:
- Verify bucket policy does not deny FleetWise writes
- Check S3 bucket for recent data files: `aws s3 ls s3://<bucket>/<prefix>/ --recursive | tail -20`
- Verify IAM role trust policy allows iotfleetwise.amazonaws.com to assume it

MAY:
- Check S3 bucket encryption configuration compatibility
- Review CloudTrail for S3 PutObject failures from FleetWise role
- Verify S3 bucket is in the same region as the FleetWise campaign

## Phase 2 — Remediate

MUST:
- Ensure IAM role has s3:PutObject permission on the destination bucket and prefix
- Verify IAM role trust policy: `{"Service": "iotfleetwise.amazonaws.com"}` in AssumeRolePolicyDocument
- Ensure S3 bucket exists in the same region as the campaign

SHOULD:
- Configure bucket lifecycle rules for data retention management
- Verify bucket encryption settings are compatible (SSE-S3 or SSE-KMS with correct key policy)
- Set appropriate S3 prefix for data organization

MAY:
- Enable S3 bucket versioning for data protection
- Configure S3 event notifications for data arrival monitoring
- Set up Athena or Glue for querying collected data

## Safety Ratings

```
safety_ratings:
  - "get-campaign: GREEN — read-only inspection"
  - "s3api head-bucket: GREEN — read-only bucket check"
  - "s3 ls: GREEN — read-only listing"
  - "iam get-role-policy: GREEN — read-only policy inspection"
  - "s3api put-bucket-policy: RED — modifies bucket access controls"
```

## Escalation Conditions

- S3 bucket policy managed by another team and cannot be modified
- KMS key policy does not allow FleetWise role to encrypt data
- S3 bucket in different account requires cross-account access setup
- Data delivery failures causing data loss for compliance-required vehicle data

## Data Sensitivity

- `aws iotfleetwise get-campaign`: MEDIUM — contains destination configuration
- `aws s3 ls`: MEDIUM — reveals data file names and sizes
- S3 data files: HIGH — contain vehicle telemetry data
- `aws iam get-role-policy`: MEDIUM — reveals permission configuration

## Prohibited Actions

- NEVER grant s3:* permissions to the FleetWise role — use least privilege (s3:PutObject)
- NEVER disable S3 bucket encryption for troubleshooting
- NEVER delete S3 data files during investigation
- NEVER modify S3 bucket policy without understanding impact on other services using the bucket

## Phase 3 — Rollback

1. If IAM role was modified, restore previous policy version
2. If bucket policy was changed, restore from backup
3. If campaign destination was changed, update campaign with correct S3 configuration
4. Verify data delivery resumes by checking for new files in S3
5. Confirm no data was lost during the outage period

## Common Issues

- symptoms: "Campaign RUNNING but no files appear in S3 bucket"
  diagnosis: "IAM role does not have s3:PutObject permission, or trust policy does not allow FleetWise to assume the role."
  resolution: "Update IAM role with s3:PutObject on s3://<bucket>/<prefix>/*. Add iotfleetwise.amazonaws.com to trust policy."

- symptoms: "S3 PutObject fails with KMS access denied"
  diagnosis: "S3 bucket uses SSE-KMS encryption and the FleetWise IAM role does not have kms:GenerateDataKey permission on the KMS key."
  resolution: "Add kms:GenerateDataKey and kms:Decrypt permissions to the FleetWise role for the bucket's KMS key."

## Output Format

```yaml
root_cause: "s3_destination — <specific_cause>"
evidence:
  - type: destination_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 destination permissions"
  long_term: "Implement destination health monitoring and data delivery alerts"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
