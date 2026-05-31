---
title: "G1 — Encryption"
description: "Diagnose WorkSpace volume encryption issues"
status: active
severity: HIGH
triggers:
  - "encryption failed"
  - "KMS key error"
  - "encrypted workspace"
  - "volume encryption"
owner: devops-agent
objective: "Resolve WorkSpace volume encryption configuration and operational issues"
context: "WorkSpaces support encryption of root and user volumes using AWS KMS. Encryption must be specified at launch time and cannot be changed after creation. The KMS key must grant the WorkSpaces service principal access."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace encryption status: `aws workspaces describe-workspaces --workspace-id <id> --query 'Workspaces[].{RootEncrypted:RootVolumeEncryptionEnabled,UserEncrypted:UserVolumeEncryptionEnabled,KmsKey:VolumeEncryptionKey}'`
- Verify KMS key exists and is enabled: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{State:KeyState,Enabled:Enabled}'`
- Check KMS key policy grants WorkSpaces access: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify KMS key is in the same region as the WorkSpace

SHOULD:
- Check for KMS key grants: `aws kms list-grants --key-id <key-id> --query 'Grants[?GranteePrincipal==\`workspaces.<region>.amazonaws.com\`]'`
- Verify IAM permissions for KMS operations
- Check CloudTrail for KMS-related errors

MAY:
- Verify KMS key rotation status
- Check if key is a customer-managed key or AWS-managed key

## Phase 2 — Remediate

MUST:
- Ensure KMS key policy allows WorkSpaces service principal (workspaces.<region>.amazonaws.com)
- Enable the KMS key if it was disabled
- Use a KMS key in the same region as the WorkSpace

SHOULD:
- Use a customer-managed KMS key for better control
- Create a KMS key grant for the WorkSpaces service if key policy is restrictive
- Document encryption requirements for new WorkSpace launches

MAY:
- Rotate to a new KMS key by launching new encrypted WorkSpaces and migrating users

## Common Issues

- symptoms: "Encrypted WorkSpace fails to launch"
  diagnosis: "KMS key policy does not grant access to WorkSpaces service principal."
  resolution: "Add WorkSpaces service principal to KMS key policy with Encrypt, Decrypt, ReEncrypt, GenerateDataKey, and DescribeKey permissions."

- symptoms: "Cannot enable encryption on existing WorkSpace"
  diagnosis: "Encryption can only be set at launch time, not modified after creation."
  resolution: "Launch a new encrypted WorkSpace and migrate user data."

- symptoms: "KMS key disabled causing WorkSpace issues"
  diagnosis: "Disabled KMS key prevents volume decryption, making WorkSpace inaccessible."
  resolution: "Re-enable the KMS key: `aws kms enable-key --key-id <key-id>`"

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<workspace encryption status and KMS key>"
  - type: kms_key_state
    content: "<KMS key state and policy>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key policy, enable key, or relaunch with encryption"
  long_term: "Standardize encryption policy and KMS key management"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- WorkSpaces contain user desktop data and files
- Directory integration reveals Active Directory structure
- Connection logs contain user access patterns
- Classification: CONFIDENTIAL - user workspace data and directory information

## Prohibited Actions

- NEVER terminate or rebuild WorkSpaces without user data backup confirmation
- NEVER modify directory service configurations during business hours
- NEVER change MFA settings without user notification and transition period
- NEVER modify VPC or subnet configurations for running WorkSpaces without maintenance window

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
