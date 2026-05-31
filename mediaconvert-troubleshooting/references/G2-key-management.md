---
title: "G2 — Key Management Issues"
description: "Diagnose encryption key management problems for MediaConvert"
status: active
severity: HIGH
triggers:
  - "key management"
  - "encryption key"
  - "KMS"
  - "content key"
owner: devops-agent
objective: "Resolve encryption key management issues for content protection and S3 encryption"
context: "Key management in MediaConvert involves two areas: DRM content keys (managed by SPEKE key provider) and S3 encryption keys (KMS for SSE-KMS encrypted inputs/outputs). Both require proper configuration."
---

## Phase 1 — Triage

MUST:
- Check job error for key-related messages: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- If KMS issue: verify key status: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify MediaConvert role has KMS permissions

SHOULD:
- Check if KMS key is in the same Region as the S3 bucket
- Verify key policy allows the MediaConvert role
- Check for key rotation status

MAY:
- Review CloudTrail for KMS access denied events
- Check SPEKE key provider key management configuration

## Phase 2 — Remediate

MUST:
- If KMS access denied: add kms:Decrypt and kms:GenerateDataKey to MediaConvert role
- If KMS key disabled: re-enable the key
- If key policy restrictive: add MediaConvert role as a key user

SHOULD:
- Use separate KMS keys for input and output encryption
- Implement key rotation
- Document key management procedures

MAY:
- Set up KMS key usage monitoring
- Implement cross-account key sharing if needed

## Common Issues

- symptoms: "Cannot read KMS-encrypted input from S3"
  diagnosis: "MediaConvert role lacks kms:Decrypt for the input bucket's KMS key."
  resolution: "Add kms:Decrypt permission and update KMS key policy."

- symptoms: "Cannot write KMS-encrypted output to S3"
  diagnosis: "MediaConvert role lacks kms:GenerateDataKey for the output bucket's KMS key."
  resolution: "Add kms:GenerateDataKey permission and update KMS key policy."

## Output Format

```yaml
root_cause: "key_management — <specific_cause>"
evidence:
  - type: kms_key
    content: "<key status and policy>"
  - type: role_permissions
    content: "<KMS permissions>"
severity: HIGH
mitigation:
  immediate: "Fix KMS permissions or key policy"
  long_term: "Implement key management best practices"
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
