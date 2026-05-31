---
title: "E1 — S3 Input/Output Issues"
description: "Diagnose S3 access problems for MediaConvert input and output"
status: active
severity: HIGH
triggers:
  - "S3 access"
  - "input bucket"
  - "output bucket"
  - "error 1040"
owner: devops-agent
objective: "Resolve S3 access issues for MediaConvert job inputs and outputs"
context: "MediaConvert reads inputs from and writes outputs to S3. Access requires both IAM role permissions and S3 bucket policy. Cross-account access needs explicit configuration. KMS-encrypted objects need additional permissions."
---

## Phase 1 — Triage

MUST:
- Check job error for S3 details: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Verify input file: `aws s3 ls <input-uri>`
- Check IAM role: `aws iam get-role --role-name <role-name>`
- Check role policy for S3 permissions: `aws iam list-attached-role-policies --role-name <role-name>`

SHOULD:
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify KMS permissions if bucket uses SSE-KMS
- Check for S3 Block Public Access settings

MAY:
- Test S3 access by assuming the MediaConvert role
- Check for VPC endpoint policies if using S3 VPC endpoints

## Phase 2 — Remediate

MUST:
- Add s3:GetObject for input bucket to the MediaConvert role
- Add s3:PutObject for output bucket to the MediaConvert role
- If KMS: add kms:Decrypt (input) and kms:GenerateDataKey (output) permissions
- Ensure role trust policy allows mediaconvert.amazonaws.com

SHOULD:
- Use resource-level permissions (specific bucket ARNs)
- For cross-account: add bucket policy allowing the MediaConvert role
- Test access before submitting batch jobs

MAY:
- Implement S3 lifecycle policies for output management
- Set up S3 event notifications for job triggering

## Common Issues

- symptoms: "Error 1040: Access Denied reading input"
  diagnosis: "IAM role lacks s3:GetObject or bucket policy denies access."
  resolution: "Add s3:GetObject to role. Check bucket policy for explicit denies."

- symptoms: "Error writing output to S3"
  diagnosis: "IAM role lacks s3:PutObject for the output bucket."
  resolution: "Add s3:PutObject permission for the output bucket ARN."

## Output Format

```yaml
root_cause: "s3_access — <specific_cause>"
evidence:
  - type: iam_role
    content: "<role permissions>"
  - type: bucket_policy
    content: "<S3 bucket policy>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role or bucket policy"
  long_term: "Implement least-privilege S3 access and testing"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or modify a production job queue while jobs are actively processing
- **NEVER** change IAM roles on job templates without verifying S3 input/output bucket access
- **NEVER** disable DRM or content protection settings on outputs destined for licensed content
- **NEVER** cancel reserved transcoding capacity without assessing active workload commitments
- **NEVER** modify output presets for compliance-regulated content without re-validation

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
