---
title: "J2 — Legal Hold Issues"
description: "Diagnose S3 Object Lock legal hold configuration and removal issues"
status: active
severity: MEDIUM
triggers:
  - "Legal hold"
  - "Cannot delete — legal hold"
  - "s3:PutObjectLegalHold"
owner: devops-agent
objective: "Identify and resolve legal hold issues on S3 objects"
context: "Legal hold is an on/off flag on an object version that prevents deletion regardless of retention settings. It requires s3:PutObjectLegalHold permission to enable or remove. Legal hold has no expiration — it must be explicitly removed. It works independently of retention mode."
---

## Phase 1 — Triage

MUST:
- Check legal hold status: `aws s3api get-object-legal-hold --bucket <bucket> --key <key>`
- Check if Object Lock is enabled on the bucket: `aws s3api get-object-lock-configuration --bucket <bucket>`
- Verify the principal has s3:PutObjectLegalHold permission to modify the hold
- Check if the object also has a retention period

SHOULD:
- Identify who placed the legal hold (CloudTrail)
- Determine if the legal hold is still required
- Check if multiple versions have legal holds

MAY:
- Audit all objects with legal holds using S3 Inventory
- Review organizational legal hold policies

## Phase 2 — Remediate

MUST:
- To remove legal hold: `aws s3api put-object-legal-hold --bucket <bucket> --key <key> --legal-hold '{"Status":"OFF"}'`
- Ensure the principal has s3:PutObjectLegalHold permission
- Verify legal hold removal is authorized (legal/compliance approval)

SHOULD:
- Document the reason for placing or removing legal holds
- Check if the object also has retention that would still prevent deletion
- Remove legal holds from all relevant versions if needed

MAY:
- Use S3 Batch Operations to manage legal holds at scale
- Implement an approval workflow for legal hold changes

## Common Issues

- symptoms: "Cannot delete object even after retention expired"
  diagnosis: "Object has a legal hold that is independent of retention."
  resolution: "Remove the legal hold with put-object-legal-hold Status=OFF."

- symptoms: "Cannot place legal hold on object"
  diagnosis: "Object Lock is not enabled on the bucket, or missing s3:PutObjectLegalHold permission."
  resolution: "Object Lock must be enabled at bucket creation. Add the IAM permission."

- symptoms: "Legal hold removed but still cannot delete"
  diagnosis: "Object also has an active retention period."
  resolution: "Check retention with get-object-retention. Both legal hold AND retention must allow deletion."

## Output Format

```yaml
root_cause: "legal_hold — <specific_cause>"
evidence:
  - type: legal_hold_status
    content: "<legal hold ON/OFF>"
  - type: retention_status
    content: "<retention mode and date if applicable>"
severity: MEDIUM
mitigation:
  immediate: "Remove legal hold if authorized"
  long_term: "Implement legal hold management process and audit with S3 Inventory"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🔴 RED | Involves modifying legal hold status via put-object-legal-hold. Removing legal holds can expose protected data to deletion. Legal hold changes have compliance and legal implications. Uses get-object-legal-hold and get-object-lock-configuration for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Legal hold changes require legal/compliance team approval

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Legal hold status reveals compliance and litigation posture"
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
