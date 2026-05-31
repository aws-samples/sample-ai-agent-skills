---
title: "G2 — Key Quota Issues"
description: "Diagnose KMS key count and resource quota problems"
status: active
severity: MEDIUM
triggers:
  - "key limit"
  - "too many keys"
  - "quota exceeded"
  - "cannot create key"
  - "LimitExceededException"
owner: devops-agent
objective: "Manage KMS key quotas and optimize key usage"
context: "KMS has per-account, per-region quotas for the number of keys, aliases, and grants. The default key limit is typically 100,000 per account per region. Grant limit is 50,000 per key. Alias limit is tied to key count. Exceeding quotas prevents new key or grant creation."
---

## Phase 1 — Triage

MUST:
- Check key count: `aws kms list-keys --query 'length(Keys)'`
- Check service quotas: `aws service-quotas get-service-quota --service-code kms --quota-code L-C2F1777E`
- Check grant count on specific keys: `aws kms list-grants --key-id <key-id> --query 'length(Grants)'`
- Check alias count: `aws kms list-aliases --query 'length(Aliases)'`

SHOULD:
- Identify unused or disabled keys that can be scheduled for deletion
- Check for keys created by AWS services that may be unnecessary
- Review grant accumulation on frequently-used keys

MAY:
- Check for keys in PendingDeletion state (still count against quota until deleted)
- Review key creation patterns in CloudTrail

## Phase 2 — Remediate

MUST:
- Schedule deletion of unused keys (after confirming no data depends on them)
- Clean up orphaned grants on keys
- Request a quota increase if legitimate usage requires more keys

SHOULD:
- Consolidate keys where possible (use fewer keys with broader policies)
- Implement key lifecycle management (creation, rotation, deletion)
- Monitor key count trends

MAY:
- Use tags to track key ownership and purpose
- Implement automated key cleanup for development/test environments

## Common Issues

- symptoms: "LimitExceededException when creating a new KMS key"
  diagnosis: "Account has reached the maximum number of KMS keys in the region."
  resolution: "Delete unused keys or request a quota increase."

- symptoms: "Cannot create grants — 50,000 grant limit per key"
  diagnosis: "A key has accumulated too many grants (common with EBS volumes)."
  resolution: "Revoke orphaned grants. Implement grant cleanup automation."

## Output Format

```yaml
root_cause: "key_quota_issue — <specific_cause>"
evidence:
  - type: key_count
    content: "<current key count vs quota>"
  - type: grant_count
    content: "<grant count on affected keys>"
severity: MEDIUM
mitigation:
  immediate: "Clean up unused keys and orphaned grants"
  long_term: "Implement key lifecycle management and monitoring"
```


## Safety Ratings

- `list-keys`: GREEN — Read-only enumeration of KMS key IDs
- `list-aliases`: GREEN — Read-only enumeration of key aliases
- `list-grants`: GREEN — Read-only enumeration of grants per key
- `get-service-quota`: GREEN — Read-only retrieval of account quota limits
- `describe-key`: GREEN — Read-only key metadata retrieval
- `request-service-quota-increase`: GREEN — Quota increase requests are reviewed by AWS
- `schedule-key-deletion`: RED — Irreversible after waiting period; all encrypted data becomes permanently inaccessible
- `revoke-grant`: YELLOW — Revoking a grant immediately removes access for the grantee

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Account is approaching key creation limits and new keys are urgently needed

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- MEDIUM: "Key metadata reveals encryption strategy"
- MEDIUM: "Key inventory reveals all KMS key IDs in the account/region"

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no active usage
- NEVER suggest creating a new key to bypass policy issues (data becomes inaccessible)

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
  - command: "get-key-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-key"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-grants"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest kms:* on key policies"
  - "NEVER suggest disabling key rotation"
  - "NEVER suggest making key policies public"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
