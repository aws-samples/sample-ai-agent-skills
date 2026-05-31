---
title: "Z1 — General KMS Troubleshooting (Catch-All)"
description: "Fallback SOP for KMS issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown KMS issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Describe the key: `aws kms describe-key --key-id <key-id>`
- Get the key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check key state: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`
- List grants: `aws kms list-grants --key-id <key-id>`
- Check CloudTrail for recent KMS events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`
- Check for throttling: `aws cloudwatch get-metric-statistics --namespace AWS/KMS --metric-name ThrottleCount --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- List all keys: `aws kms list-keys`
- Check key rotation status: `aws kms get-key-rotation-status --key-id <key-id>`
- List aliases: `aws kms list-aliases --key-id <key-id>`
- Check service quotas: `aws service-quotas get-service-quota --service-code kms --quota-code L-6E3AF000`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Access denied or policy issues → Key Policy (A1-A3)
- Grant-related problems → Grants (B1-B2)
- Cross-account access → Cross-Account (C1-C2)
- Key state issues (disabled, pending deletion) → Key State (D1-D3)
- Encryption operation failures → Encryption (E1-E3)
- Service integration errors → Integration (F1-F3)
- Throttling or quota issues → Quotas (G1-G2)
- Audit or policy conflict → Security (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all condition keys in the key policy
- Verify the principal's IAM policies and permission boundaries
- Check for SCPs that may affect KMS access
- Review the specific error message and CloudTrail event details

SHOULD:
- Simulate the policy evaluation: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <kms-action> --resource-arns <key-arn>`
- Check for VPC endpoint policies
- Verify the key spec and usage match the attempted operation

MAY:
- Check custom key store status if applicable
- Review multi-region key configuration

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<key_policy|grants|cross_account|key_state|encryption|integration|quotas|security|unknown>"
investigation_path: "key metadata → key policy → grants → key state → <domain_classification>"
evidence:
  - type: key_metadata
    content: "<key description>"
  - type: key_policy
    content: "<key policy summary>"
  - type: grants
    content: "<grant summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

- `describe-key`: GREEN — Read-only key metadata and state retrieval
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-grants`: GREEN — Read-only enumeration of grants
- `list-keys`: GREEN — Read-only enumeration of KMS key IDs
- `list-aliases`: GREEN — Read-only enumeration of key aliases
- `get-key-rotation-status`: GREEN — Read-only rotation status check
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `get-metric-statistics` (CloudWatch): GREEN — Read-only throttle metrics analysis
- `simulate-principal-policy`: GREEN — Read-only permission evaluation
- `put-key-policy`: RED — Key policy is the primary access control; incorrect changes can lock out all principals
- `schedule-key-deletion`: RED — Irreversible after waiting period; all encrypted data becomes permanently inaccessible
- `disable-key`: RED — Disabling a key blocks all cryptographic operations for every dependent service
- `enable-key`: YELLOW — Re-enables encryption/decryption for all principals with access
- `create-grant`: YELLOW — Grants provide temporary access to KMS keys; ensure scope is limited

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Issue involves production encryption keys
- Issue cannot be categorized into a specific runbook after initial triage

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
