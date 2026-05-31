---
title: "C2 — ViaService Condition Issues"
description: "Diagnose kms:ViaService condition key failures"
status: active
severity: MEDIUM
triggers:
  - "ViaService"
  - "kms:ViaService"
  - "service condition"
  - "only through service"
owner: devops-agent
objective: "Configure and troubleshoot kms:ViaService conditions correctly"
context: "The kms:ViaService condition key restricts KMS key usage to requests made through a specific AWS service (e.g., s3.us-east-1.amazonaws.com). This prevents direct KMS API calls while allowing service-integrated encryption. The condition value format is service.region.amazonaws.com."
---

## Phase 1 — Triage

MUST:
- Get the key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Identify kms:ViaService conditions in the policy
- Verify the condition value format: `<service>.<region>.amazonaws.com`
- Check if the request is coming through the expected service

SHOULD:
- Check CloudTrail for the denied event to see the actual ViaService value
- Verify the region in the condition matches the request region

MAY:
- Test with a direct KMS API call to confirm the condition blocks it

## Phase 2 — Remediate

MUST:
- Fix the ViaService condition value to match the correct service and region
- Ensure the service name format is correct (e.g., `s3.us-east-1.amazonaws.com`, `ebs.us-west-2.amazonaws.com`)

SHOULD:
- Use StringEquals for exact service matching
- Document which services are allowed to use the key

MAY:
- Use multiple ViaService values for keys used by multiple services
- Add a separate statement without ViaService for administrative operations

## Common Issues

- symptoms: "Access denied when S3 tries to use the KMS key"
  diagnosis: "ViaService condition specifies wrong region or service name."
  resolution: "Use `s3.<region>.amazonaws.com` matching the S3 bucket's region."

- symptoms: "Direct KMS API calls denied but service-integrated calls work"
  diagnosis: "This is expected behavior — ViaService restricts to service-only access."
  resolution: "Add a separate policy statement without ViaService for direct API access if needed."

## Output Format

```yaml
root_cause: "viaservice_condition_issue — <specific_cause>"
evidence:
  - type: key_policy
    content: "<policy with ViaService conditions>"
  - type: cloudtrail
    content: "<denied event showing actual context>"
severity: MEDIUM
mitigation:
  immediate: "Fix ViaService condition value"
  long_term: "Document service access patterns for each key"
```


## Safety Ratings

- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `describe-key`: GREEN — Read-only key metadata retrieval
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `put-key-policy` (ViaService condition): YELLOW — Changes affect which AWS services can use the key on behalf of principals

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
