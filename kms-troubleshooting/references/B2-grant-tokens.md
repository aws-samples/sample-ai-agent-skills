---
title: "B2 — Grant Token Issues"
description: "Diagnose grant token usage and propagation delay problems"
status: active
severity: MEDIUM
triggers:
  - "grant token"
  - "grant not effective"
  - "grant propagation"
  - "eventual consistency"
owner: devops-agent
objective: "Correctly use grant tokens for immediate access after grant creation"
context: "KMS grants may take up to 5 minutes to propagate. Grant tokens provide immediate access by proving the grant exists. The token is returned by create-grant and must be passed to subsequent KMS operations. AWS services handle grant tokens automatically."
---

## Phase 1 — Triage

MUST:
- Verify the grant was created: `aws kms list-grants --key-id <key-id> --grantee-principal <principal-arn>`
- Check if the grant token is being passed to KMS operations
- Verify the grant operations match the attempted KMS action
- Check the time elapsed since grant creation (propagation takes up to 5 minutes)

SHOULD:
- Review the application code for grant token handling
- Check CloudTrail for the create-grant event and subsequent operation attempts

MAY:
- Test with a simple encrypt/decrypt operation using the grant token

## Phase 2 — Remediate

MUST:
- Pass the grant token to KMS operations for immediate use after grant creation
- Wait up to 5 minutes if grant tokens are not being used
- Verify the grant's operations list includes the required KMS action

SHOULD:
- Implement grant token passing in application code for time-sensitive operations
- Use key policy or IAM policy for permanent access instead of grants

MAY:
- Cache grant tokens for reuse within the propagation window

## Common Issues

- symptoms: "Access denied immediately after creating a grant"
  diagnosis: "Grant has not propagated yet and no grant token is being used."
  resolution: "Pass the grant token from create-grant to the KMS operation, or wait up to 5 minutes."

- symptoms: "Grant token is invalid or expired"
  diagnosis: "Grant tokens are single-use or the grant was revoked."
  resolution: "Create a new grant and use the fresh grant token."

## Output Format

```yaml
root_cause: "grant_token_issue — <specific_cause>"
evidence:
  - type: grant
    content: "<grant details and creation time>"
  - type: operation
    content: "<KMS operation attempted>"
severity: MEDIUM
mitigation:
  immediate: "Use grant tokens or wait for propagation"
  long_term: "Implement proper grant token handling in application code"
```


## Safety Ratings

- `list-grants`: GREEN — Read-only enumeration of grants and their details
- `describe-key`: GREEN — Read-only key metadata retrieval
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `create-grant`: YELLOW — Creates temporary access; grant token returned for immediate use

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
