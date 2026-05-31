---
title: "A1 — Device Ordering Issues"
description: "Diagnose AWS Snow Family device ordering and job creation failures"
status: active
severity: HIGH
triggers: ["cannot order device", "job creation failed", "Snowball order error"]
owner: devops-agent
objective: "Resolve device ordering issues including job creation, address validation, and service availability"
context: "Snow Family jobs are created via AWS console or API. Requires valid shipping address, IAM permissions, and service availability in the region."
---

## Phase 1 — Triage

MUST:
- Check existing jobs: `aws snowball list-jobs --query 'JobListEntries[*].{JobId:JobId,State:JobState,Type:JobType}'`
- Verify IAM permissions for snowball:CreateJob
- Confirm Snow Family service is available in the target region

SHOULD:
- Verify shipping address is in a supported country and region
- Check service quotas for concurrent jobs: `aws service-quotas get-service-quota --service-code snowball --quota-code L-6032BEFE`

MAY:
- Review CloudTrail for CreateJob errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateJob --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for snowball:CreateJob, snowball:DescribeJob
- Ensure shipping address is valid and in a supported region
- Select correct device type for the use case (Snowball Edge Storage Optimized, Compute Optimized, or Snowcone)

SHOULD:
- Verify S3 bucket exists for import/export jobs
- Ensure KMS key is available for data encryption

MAY:
- Use AWS Pricing API to estimate job costs before ordering

## Safety Ratings

```
safety_ratings:
  - "list-jobs: GREEN — read-only enumeration of existing jobs"
  - "CreateJob: YELLOW — creates a new job and initiates device shipping"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Job creation fails with internal service errors after fixing IAM and configuration
- Shipping address is in a region where Snow Family is not available
- Service quota for concurrent jobs has been reached
- Job requires a device type not available in the target region

## Data Sensitivity

- `aws snowball list-jobs`: LOW — returns job metadata only
- `aws snowball describe-job`: MEDIUM — reveals shipping address and job configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER create production jobs without verifying the target S3 bucket and KMS key
- NEVER share job manifests or unlock codes outside the authorized team
- NEVER order devices to personal addresses for business use
- NEVER create cluster jobs without confirming the need for 5+ devices

## Phase 3 — Rollback

1. If a job was created in error, cancel it: `aws snowball cancel-job --job-id <job-id>` (only before device ships)
2. If a cluster was created incorrectly, cancel the cluster job before devices ship
3. Verify no charges are incurred for cancelled jobs
4. Document the cancellation reason and correct configuration for retry
5. Notify the shipping team if a device was already dispatched

## Common Issues

- symptoms: "CreateJob returns InvalidResourceException for S3 bucket"
  diagnosis: "Target S3 bucket does not exist or is in a different region."
  resolution: "Create the S3 bucket in the same region as the Snow job. Verify bucket name is correct."

## Output Format

```yaml
root_cause: "device_ordering — <specific_cause>"
evidence:
  - type: ordering_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix ordering issue"
  long_term: "Standardize job templates"
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
