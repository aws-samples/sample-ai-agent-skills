---
title: "B1 — Grant Management Issues"
description: "Diagnose KMS grant creation, retirement, and revocation problems"
status: active
severity: MEDIUM
triggers:
  - "grant not working"
  - "create grant failed"
  - "retire grant"
  - "revoke grant"
  - "too many grants"
owner: devops-agent
objective: "Manage KMS grants correctly for service and programmatic access"
context: "Grants provide temporary, scoped permissions on KMS keys. AWS services (EBS, RDS, Lambda) create grants automatically. Grants can accumulate over time and hit the 50,000 per-key limit. Grant propagation may take up to 5 minutes — use grant tokens for immediate access."
---

## Phase 1 — Triage

MUST:
- List grants on the key: `aws kms list-grants --key-id <key-id>`
- Check grant count: `aws kms list-grants --key-id <key-id> --query 'length(Grants)'`
- Describe the key for grant-related limits: `aws kms describe-key --key-id <key-id>`
- Check if the caller has permission to create grants: verify `kms:CreateGrant` in key policy or IAM

SHOULD:
- List retirable grants for a principal: `aws kms list-retirable-grants --retiring-principal <principal-arn>`
- Check for orphaned grants (grantee principal no longer exists)
- Review CloudTrail for grant operations: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateGrant --max-results 10`

MAY:
- Check for grants with constraints (encryption context)
- Identify which AWS services created grants automatically

## Phase 2 — Remediate

MUST:
- Revoke orphaned grants where the grantee no longer exists
- Fix grant creation permissions if create-grant is failing
- Use grant tokens for immediate access after grant creation

SHOULD:
- Implement grant cleanup automation for services that create grants
- Monitor grant count to prevent hitting the 50,000 limit
- Use retiring principal to allow grantees to clean up their own grants

MAY:
- Replace long-lived grants with key policy statements for permanent access
- Set up CloudWatch alarms for grant count thresholds

## Common Issues

- symptoms: "LimitExceededException — too many grants on key"
  diagnosis: "The key has reached the 50,000 grant limit."
  resolution: "Revoke unused and orphaned grants. Implement grant cleanup automation."

- symptoms: "Grant created but access not working immediately"
  diagnosis: "Grant propagation takes up to 5 minutes."
  resolution: "Use the grant token returned by create-grant for immediate access."

- symptoms: "Cannot retire grant — access denied"
  diagnosis: "The caller is not the retiring principal or grantee principal."
  resolution: "Use revoke-grant with key administrator permissions instead."

## Output Format

```yaml
root_cause: "grant_management_issue — <specific_cause>"
evidence:
  - type: grants
    content: "<grant list and count>"
  - type: key_policy
    content: "<grant-related permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix grant permissions or clean up orphaned grants"
  long_term: "Implement automated grant lifecycle management"
```


## Safety Ratings

- `list-grants`: GREEN — Read-only enumeration of existing grants
- `list-retirable-grants`: GREEN — Read-only enumeration of retirable grants
- `describe-key`: GREEN — Read-only key metadata retrieval
- `create-grant`: YELLOW — Grants provide temporary access to KMS keys; ensure scope is limited
- `revoke-grant`: YELLOW — Revoking a grant immediately removes access for the grantee
- `retire-grant`: YELLOW — Retiring a grant removes access; less disruptive than revoke

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
