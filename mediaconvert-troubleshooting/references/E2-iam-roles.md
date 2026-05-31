---
title: "E2 — IAM Role Issues"
description: "Diagnose IAM role configuration problems for MediaConvert"
status: active
severity: HIGH
triggers:
  - "IAM role"
  - "service role"
  - "trust policy"
  - "permission denied"
owner: devops-agent
objective: "Resolve IAM role configuration issues for MediaConvert operations"
context: "MediaConvert requires an IAM role with a trust policy for mediaconvert.amazonaws.com. The role needs S3 access for inputs/outputs, and optionally KMS, API Gateway (for SPEKE), and CloudWatch permissions."
---

## Phase 1 — Triage

MUST:
- Check role exists: `aws iam get-role --role-name <role-name>`
- Verify trust policy: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Check inline policies: `aws iam list-role-policies --role-name <role-name>`

SHOULD:
- Verify trust policy allows mediaconvert.amazonaws.com
- Check for permission boundaries
- Review CloudTrail for AssumeRole failures

MAY:
- Use IAM Policy Simulator to test permissions
- Check for SCPs restricting the role

## Phase 2 — Remediate

MUST:
- If trust policy wrong: update to allow mediaconvert.amazonaws.com as principal
- If missing S3 permissions: add s3:GetObject and s3:PutObject
- If missing KMS permissions: add kms:Decrypt, kms:GenerateDataKey

SHOULD:
- Use a dedicated role for MediaConvert (not shared with other services)
- Follow least-privilege with specific bucket ARNs
- Document the role configuration

MAY:
- Implement role rotation and auditing
- Use AWS managed policies where available

## Common Issues

- symptoms: "Job fails immediately with role error"
  diagnosis: "Trust policy does not allow mediaconvert.amazonaws.com."
  resolution: "Update trust policy: {\"Service\": \"mediaconvert.amazonaws.com\"} as principal."

- symptoms: "Job fails with AccessDenied on specific S3 operation"
  diagnosis: "Role policy missing the specific S3 action for the bucket."
  resolution: "Add the missing S3 action with the correct resource ARN."

## Output Format

```yaml
root_cause: "iam_role — <specific_cause>"
evidence:
  - type: trust_policy
    content: "<role trust policy>"
  - type: permissions
    content: "<attached policies>"
severity: HIGH
mitigation:
  immediate: "Fix trust policy or permissions"
  long_term: "Implement dedicated MediaConvert role with least-privilege"
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
