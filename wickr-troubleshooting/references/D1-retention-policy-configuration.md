---
title: "D1 — Retention Policy Configuration"
description: "Diagnose message retention policy configuration issues"
status: active
severity: HIGH
triggers: ["retention not working", "messages not retained", "retention policy error", "S3 retention failed"]
owner: devops-agent
objective: "Resolve message retention policy configuration and storage issues"
context: "Message retention requires explicit configuration with an S3 bucket and KMS key. Messages are ephemeral by default."
---

## Phase 1 — Triage

MUST:
- Verify network retention settings: `aws wickr get-network --network-id <id>`
- Check S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <wickr-retention-bucket>`
- Verify KMS key is enabled: `aws kms describe-key --key-id <wickr-kms-key-id>`

SHOULD:
- Check S3 bucket policy allows Wickr service access: `aws s3api get-bucket-policy --bucket <wickr-retention-bucket>`
- Verify KMS key policy grants Wickr encrypt/decrypt permissions: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Review CloudTrail for retention-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`

MAY:
- Check S3 bucket for retained message objects: `aws s3 ls s3://<wickr-retention-bucket>/ --recursive --summarize`
- Verify S3 bucket encryption configuration

## Phase 2 — Remediate

MUST:
- Configure S3 bucket policy to allow Wickr service principal access
- Ensure KMS key policy grants Wickr service encrypt and decrypt permissions
- Verify retention is enabled in the Wickr admin console

SHOULD:
- Enable S3 bucket versioning for retained message protection
- Configure S3 lifecycle rules for retention period management
- Set up CloudWatch alarms for retention failures

MAY:
- Enable S3 access logging for audit trail
- Configure cross-region replication for disaster recovery

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "s3api head-bucket: GREEN — read-only bucket check"
  - "kms describe-key: GREEN — read-only key details"
  - "s3api put-bucket-policy: YELLOW — modifies bucket access policy"
```

## Escalation Conditions

- Retained messages are missing from S3 bucket
- KMS key is disabled or scheduled for deletion
- S3 bucket is deleted or inaccessible
- Retention configuration changes do not take effect

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- `aws s3api get-bucket-policy`: MEDIUM — reveals access control configuration
- `aws kms get-key-policy`: MEDIUM — reveals encryption key access
- Retained messages in S3: HIGH — contains encrypted message data

## Prohibited Actions

- NEVER delete the retention S3 bucket without backing up retained messages
- NEVER disable or delete the KMS key used for retention encryption
- NEVER access retained message content without proper authorization
- NEVER modify retention policies without compliance team approval

## Phase 3 — Rollback

1. If S3 bucket policy was modified, restore original policy
2. If KMS key policy was changed, restore original key policy
3. If retention was disabled, re-enable with original configuration
4. Verify new messages are being retained after rollback
5. Document retention configuration changes for compliance audit

## Common Issues

- symptoms: "Retention enabled but no messages appearing in S3"
  diagnosis: "S3 bucket policy does not grant Wickr service access."
  resolution: "Update S3 bucket policy to allow the Wickr service principal to write objects."

- symptoms: "Retention fails with KMS access denied"
  diagnosis: "KMS key policy does not grant Wickr service encrypt/decrypt permissions."
  resolution: "Update KMS key policy to allow Wickr service principal kms:GenerateDataKey and kms:Decrypt."

## Output Format

```yaml
root_cause: "retention_policy — <specific_cause>"
evidence:
  - type: retention_configuration
    content: "<config details>"
severity: HIGH
mitigation:
  immediate: "Fix retention configuration"
  long_term: "Implement retention monitoring and compliance checks"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
