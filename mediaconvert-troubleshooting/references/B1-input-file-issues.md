---
title: "B1 — Input File Issues"
description: "Diagnose input file access and format problems"
status: active
severity: HIGH
triggers:
  - "input file"
  - "cannot read input"
  - "input access"
  - "source file"
owner: devops-agent
objective: "Resolve input file access, format, and compatibility issues"
context: "MediaConvert reads input files from S3. Issues include S3 access denied, corrupted files, unsupported formats, and incorrect file paths. Error code 1000 indicates input-related failures."
---

## Phase 1 — Triage

MUST:
- Verify input file exists: `aws s3 ls <input-s3-uri>`
- Check file size: `aws s3 ls <input-s3-uri> --human-readable`
- Verify IAM role can access the file: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns <s3-arn>`
- Check job error for input-specific messages

SHOULD:
- Verify S3 bucket policy allows MediaConvert role access
- Check if file is encrypted (SSE-KMS requires additional permissions)
- Test downloading the file to verify it's not corrupted

MAY:
- Use MediaInfo to analyze input file properties
- Check for S3 versioning issues (wrong version)

## Phase 2 — Remediate

MUST:
- If access denied: fix IAM role policy or S3 bucket policy
- If KMS encrypted: add kms:Decrypt permission to the MediaConvert role
- If file corrupted: re-upload the source file
- If wrong path: correct the S3 URI in the job settings

SHOULD:
- Use S3 bucket policies that explicitly allow the MediaConvert role
- Test input access before submitting large batch jobs
- Validate input files before processing

MAY:
- Implement input validation pipeline
- Use S3 event notifications to trigger jobs on upload

## Common Issues

- symptoms: "Error 1000: Unable to open input file"
  diagnosis: "S3 path incorrect or IAM role lacks s3:GetObject permission."
  resolution: "Verify S3 URI. Add s3:GetObject to the MediaConvert role for the input bucket."

- symptoms: "Error reading KMS-encrypted input"
  diagnosis: "MediaConvert role lacks kms:Decrypt for the KMS key."
  resolution: "Add kms:Decrypt and kms:DescribeKey permissions for the KMS key ARN."

## Output Format

```yaml
root_cause: "input_file — <specific_cause>"
evidence:
  - type: s3_access
    content: "<S3 file verification>"
  - type: iam_permissions
    content: "<role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix file access or re-upload source"
  long_term: "Implement input validation and access testing"
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
