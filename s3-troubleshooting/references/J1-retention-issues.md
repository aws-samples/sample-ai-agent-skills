---
title: "J1 — Object Lock Retention Issues"
description: "Diagnose S3 Object Lock retention mode and period issues"
status: active
severity: MEDIUM
triggers:
  - "Object Lock"
  - "Retention period"
  - "Cannot delete object"
  - "Governance mode"
  - "Compliance mode"
  - "WORM"
owner: devops-agent
objective: "Identify and resolve Object Lock retention configuration issues"
context: "S3 Object Lock provides WORM (Write Once Read Many) protection. Two retention modes: Governance (can be overridden with s3:BypassGovernanceRetention) and Compliance (cannot be overridden by anyone, including root). Object Lock must be enabled at bucket creation. Retention can be set per-object or as bucket defaults."
---

## Phase 1 — Triage

MUST:
- Check if Object Lock is enabled: `aws s3api get-object-lock-configuration --bucket <bucket>`
- Check object retention: `aws s3api get-object-retention --bucket <bucket> --key <key>`
- Identify the retention mode (Governance or Compliance) and expiration date
- Check bucket-level default retention: `aws s3api get-object-lock-configuration --bucket <bucket> --query 'ObjectLockConfiguration.Rule'`

SHOULD:
- Verify the principal has s3:BypassGovernanceRetention if trying to override Governance mode
- Check if the retention period has expired
- Verify versioning is enabled (required for Object Lock)

MAY:
- Check legal hold status: `aws s3api get-object-legal-hold --bucket <bucket> --key <key>`
- Review CloudTrail for retention-related operations

## Phase 2 — Remediate

MUST:
- For Governance mode override: use `--bypass-governance-retention` flag with appropriate IAM permission
- For Compliance mode: wait for the retention period to expire (cannot be shortened)
- To extend retention: `aws s3api put-object-retention --bucket <bucket> --key <key> --retention '{"Mode":"GOVERNANCE","RetainUntilDate":"<date>"}'`

SHOULD:
- Set appropriate bucket-level defaults to avoid per-object configuration
- Use Governance mode during testing, Compliance mode for regulatory requirements
- Document retention policies and expiration dates

MAY:
- Use S3 Inventory to audit retention settings across all objects
- Implement lifecycle rules that work alongside Object Lock (transitions are allowed)

## Common Issues

- symptoms: "Cannot delete object — Access Denied"
  diagnosis: "Object has active retention period that has not expired."
  resolution: "For Governance: use BypassGovernanceRetention. For Compliance: wait for expiration."

- symptoms: "Cannot shorten retention period"
  diagnosis: "Retention periods can only be extended, never shortened (both modes)."
  resolution: "Wait for the current retention to expire. Plan retention periods carefully."

- symptoms: "Object Lock not available on existing bucket"
  diagnosis: "Object Lock can only be enabled at bucket creation time."
  resolution: "Create a new bucket with Object Lock enabled and migrate objects."

## Output Format

```yaml
root_cause: "retention_issue — <specific_cause>"
evidence:
  - type: object_lock_config
    content: "<Object Lock configuration>"
  - type: object_retention
    content: "<retention mode and date>"
severity: MEDIUM
mitigation:
  immediate: "Override Governance retention or wait for Compliance expiration"
  long_term: "Set appropriate bucket defaults and document retention policies"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🔴 RED | Involves modifying Object Lock retention settings via put-object-retention and potentially bypassing Governance mode. Incorrect retention changes can make data permanently undeletable (Compliance mode) or expose data to premature deletion. Uses get-object-lock-configuration and get-object-retention for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Retention changes affect regulatory compliance or legal hold requirements

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Object Lock configuration reveals compliance and retention strategy"
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
