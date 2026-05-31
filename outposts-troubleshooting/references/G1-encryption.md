---
title: "G1 — Outpost Encryption Issues"
description: "Diagnose encryption configuration problems on AWS Outposts"
status: active
severity: HIGH
triggers:
  - "outpost encryption"
  - "KMS outpost"
  - "encrypted volume outpost"
  - "data at rest outpost"
owner: devops-agent
objective: "Resolve encryption configuration and key management issues on Outposts"
context: "Outpost data at rest is encrypted by default using AWS-managed keys. Customer-managed KMS keys can be used for EBS volumes and S3 on Outposts. KMS operations require service link connectivity to the Region."
---

## Phase 1 — Triage

MUST:
- Check EBS volume encryption: `aws ec2 describe-volumes --filters "Name=outpost-arn,Values=<outpost-arn>" --query 'Volumes[*].{Id:VolumeId,Encrypted:Encrypted,KmsKeyId:KmsKeyId}'`
- Verify KMS key status: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify service link connectivity (KMS calls require it)

SHOULD:
- Check if default EBS encryption is enabled: `aws ec2 get-ebs-encryption-by-default`
- Verify IAM permissions for KMS operations
- Check S3 on Outposts bucket encryption configuration

MAY:
- Review KMS key rotation status: `aws kms get-key-rotation-status --key-id <key-id>`
- Check CloudTrail for KMS-related errors

## Phase 2 — Remediate

MUST:
- If KMS key disabled: re-enable the key: `aws kms enable-key --key-id <key-id>`
- If permission denied: update KMS key policy to allow the required principals
- Ensure service link is up for KMS operations

SHOULD:
- Enable default EBS encryption for the account in the Outpost Region
- Use customer-managed KMS keys for compliance requirements
- Implement key rotation policies

MAY:
- Set up KMS key usage monitoring with CloudTrail
- Implement cross-account key sharing if needed

## Common Issues

- symptoms: "Cannot create encrypted volume on Outpost"
  diagnosis: "KMS key policy does not allow the EC2 service or the user's IAM role."
  resolution: "Update KMS key policy to grant kms:CreateGrant and kms:Decrypt to the required principals."

- symptoms: "Encrypted volume operations fail during service link outage"
  diagnosis: "KMS operations require Region connectivity via the service link."
  resolution: "Restore service link. Existing encrypted volumes remain accessible but new operations fail."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: volume_encryption
    content: "<volume encryption status>"
  - type: kms_key
    content: "<KMS key status and policy>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key policy or restore service link"
  long_term: "Implement default encryption and key management policies"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — encryption protects data at rest and in transit
- Service Disruption Risk: MEDIUM — encryption key issues can prevent data access
- Compliance Impact: HIGH — encryption is a fundamental compliance requirement

## Escalation Conditions
- Encryption key becomes inaccessible
- Encryption configuration changes affect data access
- KMS key rotation fails
- Unencrypted data discovered on Outpost
- Encryption audit reveals compliance gaps

## Data Sensitivity
- KMS key IDs and configurations — CONFIDENTIAL
- Encryption policies — INTERNAL
- Key rotation schedules — INTERNAL
- Encryption audit results — CONFIDENTIAL
- Never expose KMS key material or key policies

## Prohibited Actions
- Do NOT disable encryption on production Outpost resources
- Do NOT delete KMS keys without verifying no resources depend on them
- Do NOT share KMS key access with unauthorized accounts
- Do NOT bypass encryption requirements for any Outpost resource
- Do NOT ignore encryption key rotation schedules
- Do NOT store encryption keys outside of KMS

## Phase 3 — Rollback
- Restore KMS key policies if modified
- Re-enable encryption if accidentally disabled
- Restore key rotation schedules
- Verify all resources are encrypted after changes
- Audit encryption compliance after any key changes
- Document encryption configuration changes and their impact

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
