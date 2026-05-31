---
title: "H2 — Encryption Issues"
description: "Diagnose encryption configuration problems for Storage Gateway"
status: active
severity: MEDIUM
triggers:
  - "encryption error"
  - "KMS key"
  - "data encryption"
  - "SSE configuration"
owner: devops-agent
objective: "Identify and resolve encryption configuration issues for Storage Gateway"
context: "Storage Gateway supports encryption at rest using AWS KMS keys for file shares, volumes, and tapes. Data in transit is encrypted with TLS. File shares can use S3 server-side encryption (SSE-S3 or SSE-KMS). Volume snapshots are encrypted. KMS key permissions must allow the gateway's IAM role."
---

## Phase 1 — Triage

MUST:
- Check file share encryption settings: `aws storagegateway describe-nfs-file-shares --file-share-arn-list <arn>` (check KMSEncrypted and KMSKey fields)
- Verify KMS key exists and is enabled: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy allows the gateway's IAM role
- Verify gateway IAM role has kms:Decrypt and kms:GenerateDataKey permissions

SHOULD:
- Check if KMS key is in the same region as the gateway
- Verify S3 bucket default encryption settings
- Check for KMS key rotation status

MAY:
- Review KMS key usage logs in CloudTrail
- Check for KMS throttling (requests per second limits)

## Phase 2 — Remediate

MUST:
- Grant KMS permissions to gateway IAM role: kms:Decrypt, kms:GenerateDataKey, kms:DescribeKey
- Update KMS key policy to allow the gateway's IAM role
- Ensure KMS key is enabled and not pending deletion

SHOULD:
- Configure file share with KMS encryption: `aws storagegateway update-nfs-file-share --file-share-arn <arn> --kms-encrypted --kms-key <key-arn>`
- Enable KMS key rotation for compliance

MAY:
- Use separate KMS keys for different file shares/volumes
- Implement KMS key policies with conditions for enhanced security

## Common Issues

- symptoms: "Access denied when writing encrypted files"
  diagnosis: "Gateway IAM role lacks KMS permissions."
  resolution: "Grant kms:Decrypt and kms:GenerateDataKey to the gateway's IAM role on the KMS key."

- symptoms: "File share creation fails with KMS error"
  diagnosis: "KMS key not found, disabled, or in wrong region."
  resolution: "Verify KMS key ARN, ensure key is enabled, and in the same region as gateway."

- symptoms: "Encryption not applied to new files"
  diagnosis: "File share not configured for KMS encryption."
  resolution: "Update file share with --kms-encrypted flag and specify KMS key ARN."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<file share/volume encryption settings>"
  - type: kms_key
    content: "<KMS key status and policy>"
  - type: iam_permissions
    content: "<gateway role KMS permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix KMS key permissions and encryption configuration"
  long_term: "Implement encryption standards and key rotation policies"
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

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

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
