---
title: "D2 — Encryption Issues"
description: "Diagnose MediaConnect encryption configuration and key management problems"
status: active
severity: HIGH
triggers: ["encryption error", "decryption failed", "SPEKE error", "static key error", "SRT passphrase", "key rotation"]
owner: devops-agent
objective: "Resolve encryption configuration issues for sources, outputs, and entitlements"
context: "MediaConnect supports SPEKE, static-key (Secrets Manager), and SRT passphrase encryption. Each model has different configuration requirements and failure modes."
---

## Phase 1 — Triage

MUST:
- Check source encryption config: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Source.Encryption'`
- Check output encryption config: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Outputs[*].{Name:Name,Encryption:Encryption}'`
- For static-key: verify Secrets Manager secret exists: `aws secretsmanager describe-secret --secret-id <secret-arn>`

SHOULD:
- For SPEKE: verify key provider URL is reachable and IAM role has invoke permissions
- For static-key: verify secret value is valid base64-encoded 128-bit or 256-bit key
- For SRT: verify passphrase matches on both sender and receiver

MAY:
- Check KMS key state if Secrets Manager uses KMS encryption: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{KeyState:KeyState,KeyUsage:KeyUsage}'`
- Review CloudTrail for encryption-related API errors

## Phase 2 — Remediate

MUST:
- For static-key: ensure Secrets Manager secret contains valid key: `aws secretsmanager get-secret-value --secret-id <secret-arn> --query 'SecretString'`
- For SPEKE: verify IAM role trust policy allows MediaConnect to assume it
- For SRT: update passphrase to match: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --decryption Algorithm=aes256,RoleArn=<role-arn>,SecretArn=<secret-arn>`

SHOULD:
- Verify IAM role has secretsmanager:GetSecretValue permission for the specific secret ARN
- Ensure KMS key policy allows the IAM role to decrypt

MAY:
- Rotate encryption keys using Secrets Manager rotation
- Update entitlement encryption: `aws mediaconnect update-flow-entitlement --flow-arn <flow-arn> --entitlement-arn <entitlement-arn> --encryption Algorithm=aes256,RoleArn=<role-arn>,SecretArn=<secret-arn>`

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "secretsmanager describe-secret: GREEN — read-only secret metadata"
  - "secretsmanager get-secret-value: RED — retrieves actual encryption key material"
  - "update-flow-source (encryption): YELLOW — modifies encryption on potentially live source"
```

## Escalation Conditions

- SPEKE key provider endpoint is unreachable or returning errors
- KMS key used by Secrets Manager is disabled or pending deletion
- Encryption key mismatch between source and sender cannot be resolved
- Content protection compliance requires immediate encryption restoration

## Data Sensitivity

- `aws secretsmanager get-secret-value`: CRITICAL — returns actual encryption key material
- `aws kms describe-key`: LOW — returns key metadata only
- `aws mediaconnect describe-flow (encryption)`: MEDIUM — reveals encryption algorithm and role ARN
- `aws secretsmanager describe-secret`: LOW — returns secret metadata without value

## Prohibited Actions

- NEVER log or display encryption key values in plain text
- NEVER disable encryption on production flows carrying protected content
- NEVER share Secrets Manager secret ARNs or KMS key IDs in unsecured channels
- NEVER use the same encryption key across multiple unrelated flows without security review

## Phase 3 — Rollback

1. If encryption was changed incorrectly, revert source encryption: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --decryption Algorithm=<original-algo>,RoleArn=<original-role>,SecretArn=<original-secret>`
2. If Secrets Manager secret was modified, restore previous version: `aws secretsmanager get-secret-value --secret-id <secret-arn> --version-stage AWSPREVIOUS`
3. If KMS key was disabled, re-enable: `aws kms enable-key --key-id <key-id>` (requires kms:EnableKey permission)
4. Verify encryption is working by checking source and output bitrate metrics
5. Document the encryption change and notify the security team

## Common Issues

- symptoms: "Source shows encryption error, no bitrate"
  diagnosis: "Encryption key mismatch between sender and MediaConnect source decryption config."
  resolution: "Verify the Secrets Manager secret contains the same key the sender is using. Check algorithm matches (aes128 vs aes256)."

- symptoms: "SPEKE encryption fails with 403 error"
  diagnosis: "IAM role cannot invoke the SPEKE key provider URL."
  resolution: "Verify IAM role has permission to call the SPEKE endpoint. Check role trust policy allows mediaconnect.amazonaws.com."

- symptoms: "Static-key decryption fails with InvalidSecretValue"
  diagnosis: "Secrets Manager secret does not contain valid base64-encoded key of correct length."
  resolution: "Update secret with valid base64-encoded 128-bit (16 bytes) or 256-bit (32 bytes) key."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<encryption settings and error details>"
severity: HIGH
mitigation:
  immediate: "Fix encryption configuration"
  long_term: "Implement key rotation and monitoring for encryption health"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
