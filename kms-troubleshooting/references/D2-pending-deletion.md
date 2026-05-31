---
title: "D2 — Pending Deletion Recovery"
description: "Diagnose and recover keys in PendingDeletion state"
status: active
severity: CRITICAL
triggers:
  - "pending deletion"
  - "key scheduled for deletion"
  - "cancel deletion"
  - "key will be deleted"
owner: devops-agent
objective: "Cancel key deletion if possible and prevent data loss"
context: "Keys in PendingDeletion state have a 7-30 day waiting period before permanent deletion. During this period, the key cannot be used but deletion CAN be cancelled. After the waiting period, the key and all data encrypted with it are permanently unrecoverable."
---

## Phase 1 — Triage

MUST:
- Check key state and deletion date: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{State:KeyState,DeletionDate:DeletionDate}'`
- Verify the key is in PendingDeletion state (not already deleted)
- Check CloudTrail for who scheduled deletion: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ScheduleKeyDeletion --max-results 10`
- Identify all resources encrypted with this key

SHOULD:
- Check how many days remain before deletion
- Identify the business impact of losing this key

MAY:
- Check for automation that may have scheduled the deletion

## Phase 2 — Remediate

MUST:
- Cancel deletion immediately if the key is still needed: `aws kms cancel-key-deletion --key-id <key-id>`
- Re-enable the key after cancelling deletion: `aws kms enable-key --key-id <key-id>`
- Verify cryptographic operations work after recovery

SHOULD:
- Investigate why the key was scheduled for deletion
- Set up CloudWatch Events for ScheduleKeyDeletion events
- Implement approval workflows for key deletion

MAY:
- Re-encrypt data with a new key as a precaution
- Create a backup plan for key material (for imported keys)

## Common Issues

- symptoms: "Key is in PendingDeletion — services failing"
  diagnosis: "Key was scheduled for deletion and cannot be used during the waiting period."
  resolution: "Cancel deletion immediately: `aws kms cancel-key-deletion --key-id <key-id>` then enable."

- symptoms: "Cannot cancel deletion — key already deleted"
  diagnosis: "The waiting period expired and the key was permanently deleted."
  resolution: "Data encrypted with this key is permanently lost. No recovery is possible."

## Output Format

```yaml
root_cause: "pending_deletion — <specific_cause>"
evidence:
  - type: key_state
    content: "<key state and deletion date>"
  - type: cloudtrail
    content: "<ScheduleKeyDeletion event>"
severity: CRITICAL
mitigation:
  immediate: "Cancel key deletion and re-enable the key"
  long_term: "Implement deletion protection and approval workflows"
```


## Safety Ratings

- `describe-key`: GREEN — Read-only key metadata and deletion date retrieval
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `cancel-key-deletion`: RED — Time-sensitive; once the waiting period expires, the key and all encrypted data are permanently lost
- `schedule-key-deletion`: RED — Irreversible after waiting period; all encrypted data becomes permanently inaccessible
- `enable-key`: YELLOW — Re-enables key after cancelling deletion

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
