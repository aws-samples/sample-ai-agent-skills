---
title: "F2 — Encryption Issues"
description: "Diagnose and resolve encryption issues for SQS, SNS, and EventBridge"
status: active
severity: HIGH
triggers:
  - "KMS error"
  - "encryption failed"
  - "SSE-SQS"
  - "SSE-KMS"
  - "decrypt error"
  - "KMSAccessDeniedException"
owner: devops-agent
objective: "Resolve encryption-related issues affecting event-driven service operations"
context: "SQS supports SSE-SQS (AWS managed) and SSE-KMS (customer managed) encryption. SNS supports KMS encryption for messages at rest. EventBridge does not encrypt events at rest by default but targets may require encryption. Encryption issues arise from KMS key permissions, key state (disabled/pending deletion), cross-account key access, or service-linked role permissions for KMS."
---

## Phase 1 — Triage

MUST:
- Check queue/topic encryption configuration:
  - SQS: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names KmsMasterKeyId,SqsManagedSseEnabled`
  - SNS: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.KmsMasterKeyId'`
- Verify the KMS key state: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{State:KeyState,Enabled:Enabled}'`
- Check KMS key policy allows the service to use the key: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Review CloudTrail for KMS API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`
- Verify the caller has kms:Decrypt and kms:GenerateDataKey permissions

SHOULD:
- Check if the KMS key has grants for the service: `aws kms list-grants --key-id <key-id>`
- Verify the SQS/SNS service principal is in the KMS key policy
- Check if the key is in a different account and cross-account access is configured
- Review the KMS key rotation status

MAY:
- Check if the key alias resolves correctly: `aws kms describe-key --key-id alias/<alias-name>`
- Verify no SCPs are blocking KMS operations
- Review the KMS request rate against service limits

## Phase 2 — Remediate

MUST:
- If KMS key disabled: re-enable it: `aws kms enable-key --key-id <key-id>`
- If KMS key pending deletion: cancel deletion: `aws kms cancel-key-deletion --key-id <key-id>` then re-enable
- If key policy missing service permissions: add the SQS/SNS service principal to the key policy
- If caller lacks KMS permissions: add kms:Decrypt and kms:GenerateDataKey to the IAM policy

SHOULD:
- For simpler management, consider switching to SSE-SQS (AWS managed): `aws sqs set-queue-attributes --queue-url <queue-url> --attributes SqsManagedSseEnabled=true`
- Verify encryption operations work after remediation by sending and receiving a test message
- Enable KMS key rotation for security best practices

MAY:
- Implement KMS key monitoring with CloudWatch
- Set up alerts for KMS key state changes
- Document the KMS key usage across services

## Common Issues

- symptoms: "SendMessage returns KMSAccessDeniedException"
  diagnosis: "The caller or service does not have permission to use the KMS key for encryption."
  resolution: "Add kms:GenerateDataKey permission for the caller. Add the SQS service principal to the key policy."

- symptoms: "ReceiveMessage returns KMSAccessDeniedException"
  diagnosis: "The caller does not have kms:Decrypt permission on the KMS key."
  resolution: "Add kms:Decrypt permission to the caller's IAM policy for the specific KMS key."

- symptoms: "SNS cannot deliver to encrypted SQS queue"
  diagnosis: "SNS service principal does not have permission to use the SQS queue's KMS key."
  resolution: "Add kms:GenerateDataKey and kms:Decrypt for the sns.amazonaws.com principal in the KMS key policy."

- symptoms: "Encryption operations fail with KMSInvalidStateException"
  diagnosis: "KMS key is disabled or pending deletion."
  resolution: "Re-enable the key or cancel the scheduled deletion."

## Output Format

```yaml
root_cause: "Encryption issue — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<queue/topic encryption settings>"
  - type: kms_key_state
    content: "<KMS key status and policy>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key permissions or state"
  long_term: "Implement KMS key monitoring and proper key management policies"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 KMS key policy changes: RED — affects encryption key access control"
  - "Phase 2 KMS key state changes (enable/cancel deletion): RED — affects key availability"

## Escalation Conditions

escalation_conditions:
  - "KMS key is pending deletion and contains critical encryption material"
  - "Key policy changes affect multiple services using the same key"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Cross-account KMS access requires coordination between account owners"
  - "Issue involves compliance-required encryption (HIPAA, PCI DSS)"

## Data Sensitivity

data_sensitivity:
  - command: "get-key-policy"
    sensitivity: HIGH
    contains: "KMS key access control policy"
  - command: "list-grants"
    sensitivity: HIGH
    contains: "KMS key grant details and grantee principals"
  - command: "describe-key"
    sensitivity: MEDIUM
    contains: "KMS key metadata and state"

## Prohibited Actions

prohibited_actions:
  - "NEVER disable or schedule deletion of KMS keys without verifying no services depend on them"
  - "NEVER add Principal: * to KMS key policies"
  - "NEVER switch from SSE-KMS to unencrypted without compliance review"
  - "NEVER share KMS key IDs or policies with unauthorized parties"
  - "NEVER create new KMS keys when the issue is a permission problem on the existing key"

## Phase 3 — Rollback

- Before changing KMS key policies, save the current policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- If key policy changes break other services, restore the previous policy
- If encryption was changed on a queue/topic, revert to the previous encryption setting
- If a KMS key was re-enabled, verify all dependent services can use it
- Monitor KMS API errors and service operations after any changes
