---
title: "F3 — MFA Delete Issues"
description: "Diagnose MFA Delete configuration and operational issues"
status: active
severity: MEDIUM
triggers:
  - "MFA Delete"
  - "Cannot delete version"
  - "MFA required"
  - "MfaDelete"
owner: devops-agent
objective: "Identify and resolve MFA Delete configuration issues"
context: "MFA Delete adds an extra layer of protection requiring MFA authentication to delete object versions or change versioning state. Only the root account can enable/disable MFA Delete. It cannot be enabled via the console — only via CLI or API with root credentials and an MFA token."
---

## Phase 1 — Triage

MUST:
- Check if MFA Delete is enabled: `aws s3api get-bucket-versioning --bucket <bucket>`
- Look for MFADelete: Enabled in the response
- Identify who is trying to delete and whether they have MFA
- Check if the operation requires MFA (version deletion, versioning state change)

SHOULD:
- Verify the root account has an MFA device configured
- Check if the error is "MFA Delete must be used" or similar
- Determine if MFA Delete was intentionally enabled

MAY:
- Check CloudTrail for who enabled MFA Delete
- Review organizational policy on MFA Delete requirements

## Phase 2 — Remediate

MUST:
- To delete with MFA: include MFA in the request: `aws s3api delete-object --bucket <bucket> --key <key> --version-id <id> --mfa "arn:aws:iam::<account>:mfa/<device> <code>"`
- To disable MFA Delete: use root account credentials with MFA:
  `aws s3api put-bucket-versioning --bucket <bucket> --versioning-configuration Status=Enabled,MFADelete=Disabled --mfa "arn:aws:iam::<account>:mfa/<device> <code>"`
- Only the root account can enable or disable MFA Delete

SHOULD:
- Document which buckets have MFA Delete enabled
- Ensure root account MFA device is accessible for emergencies
- Consider if MFA Delete is still needed for the use case

MAY:
- Implement a break-glass procedure for MFA Delete operations
- Use Object Lock instead of MFA Delete for compliance requirements

## Common Issues

- symptoms: "Cannot delete object versions — MFA required"
  diagnosis: "MFA Delete is enabled on the bucket."
  resolution: "Include MFA token in the delete request, or disable MFA Delete using root account."

- symptoms: "Cannot enable MFA Delete with IAM user"
  diagnosis: "Only the root account can enable/disable MFA Delete."
  resolution: "Use root account credentials with MFA to manage MFA Delete."

- symptoms: "Cannot disable MFA Delete — lost MFA device"
  diagnosis: "Root account MFA device is required to disable MFA Delete."
  resolution: "Contact AWS Support to recover root account MFA access."

## Output Format

```yaml
root_cause: "mfa_delete — <specific_cause>"
evidence:
  - type: versioning_config
    content: "<MFADelete status>"
  - type: error_message
    content: "<MFA-related error>"
severity: MEDIUM
mitigation:
  immediate: "Use root account with MFA to perform the operation"
  long_term: "Document MFA Delete status and maintain root account MFA access"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying MFA Delete settings and deleting versioned objects via put-bucket-versioning and delete-object with MFA. MFA Delete changes are state-changing but recoverable (can be re-enabled). Uses get-bucket-versioning for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- MFA Delete changes affect data protection and compliance posture

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Lifecycle rules reveal data retention strategy"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
