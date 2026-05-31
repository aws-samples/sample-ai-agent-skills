---
title: "D1 — Disabled Key Issues"
description: "Diagnose issues caused by disabled KMS keys"
status: active
severity: HIGH
triggers:
  - "key is disabled"
  - "DisabledException"
  - "key state disabled"
  - "cannot encrypt"
  - "cannot decrypt"
owner: devops-agent
objective: "Identify disabled keys and restore cryptographic operations"
context: "A disabled KMS key cannot perform any cryptographic operation. All AWS services using the key will fail. Disabling is reversible (unlike deletion). Common causes include accidental disabling, security response, or automated processes."
---

## Phase 1 — Triage

MUST:
- Check key state: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`
- Verify the key is disabled (state = Disabled)
- Check CloudTrail for who disabled the key: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DisableKey --max-results 10`
- Identify services affected by the disabled key

SHOULD:
- Check if the key was disabled as part of a security incident response
- Verify the caller has permission to re-enable: `kms:EnableKey`

MAY:
- Check for automation (Lambda, Config rules) that may have disabled the key

## Phase 2 — Remediate

MUST:
- Re-enable the key if appropriate: `aws kms enable-key --key-id <key-id>`
- Verify cryptographic operations work after re-enabling
- Confirm affected services resume normal operation

SHOULD:
- Investigate why the key was disabled to prevent recurrence
- Set up CloudWatch Events for key state changes
- Document the key's purpose and authorized administrators

MAY:
- Create a CloudWatch alarm for DisableKey events
- Implement approval workflows for key state changes

## Common Issues

- symptoms: "S3 PutObject fails with KMS error after key was disabled"
  diagnosis: "The KMS key used for S3 SSE-KMS encryption is disabled."
  resolution: "Re-enable the key or change the S3 bucket encryption to use a different key."

- symptoms: "EBS volume cannot be attached — encryption key unavailable"
  diagnosis: "The KMS key used to encrypt the EBS volume is disabled."
  resolution: "Re-enable the key. EBS volumes cannot be re-encrypted with a different key."

## Output Format

```yaml
root_cause: "disabled_key — <specific_cause>"
evidence:
  - type: key_state
    content: "<key metadata showing Disabled state>"
  - type: cloudtrail
    content: "<DisableKey event details>"
severity: HIGH
mitigation:
  immediate: "Re-enable the key if appropriate"
  long_term: "Monitor key state changes and implement approval workflows"
```


## Safety Ratings

- `describe-key`: GREEN — Read-only key metadata and state retrieval
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `enable-key`: YELLOW — Re-enables encryption/decryption for all principals with access
- `disable-key`: RED — Disabling a key blocks all cryptographic operations for every dependent service

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change (disable/enable/schedule deletion) is needed
- Multiple services depend on the affected key

## Rollback

1. Before any key policy change: Save current policy with `aws kms get-key-policy`
2. After change: Verify encryption/decryption operations work
3. If change causes issues: Restore the saved key policy immediately
4. Cleanup: Revoke any temporary grants created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-key-policy` | HIGH | Contains all authorized principals — redact |
| `list-grants` | HIGH | Shows who has access — summarize |
| `describe-key` | MEDIUM | Key metadata — safe to summarize |
| `list-keys` | LOW | Key IDs only — safe to include |

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no resources depend on it
- NEVER suggest creating a new key to "work around" a policy issue
- ALWAYS identify the specific KMS action needed (kms:Decrypt, kms:GenerateDataKey, etc.)
- ALWAYS verify the encryption context matches before modifying policies

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
