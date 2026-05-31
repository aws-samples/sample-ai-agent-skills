---
title: "D3 — Cross-Region Copy Failures"
description: "Diagnose failures when copying backup recovery points to another AWS region"
status: active
severity: HIGH
triggers:
  - "cross-region copy"
  - "copy job failed"
  - "copy to another region"
  - "disaster recovery copy"
  - "backup replication"
owner: devops-agent
objective: "Identify and resolve issues with cross-region backup copy operations"
context: "Cross-region copy creates a copy of a recovery point in a backup vault in another region. It requires a destination vault with a KMS key in the destination region. Copy actions can be configured in backup plan rules or triggered on-demand. KMS key policies and IAM permissions are the most common failure points."
---

## Phase 1 — Triage

MUST:
- List failed copy jobs: `aws backup list-copy-jobs --by-state FAILED --max-results 20`
- Describe the failed copy job: `aws backup describe-copy-job --copy-job-id <copy-job-id>`
- Check the StatusMessage for the specific error
- Verify the destination vault exists in the target region: `aws backup describe-backup-vault --backup-vault-name <dest-vault> --region <dest-region>`
- Check the KMS key in the destination region: `aws kms describe-key --key-id <dest-key-id> --region <dest-region>`

SHOULD:
- Verify the IAM role has permissions for both source and destination regions
- Check the source recovery point status: `aws backup describe-recovery-point --backup-vault-name <source-vault> --recovery-point-arn <rp-arn>`
- Verify the destination vault's KMS key policy allows the backup role
- Check if the copy action in the backup plan is configured correctly

MAY:
- Check CloudTrail in both regions for copy-related events
- Verify no SCP blocks cross-region operations
- Check if the resource type supports cross-region copy

## Phase 2 — Remediate

MUST:
- Ensure a backup vault exists in the destination region with appropriate KMS encryption
- Grant the backup IAM role permissions on the destination region's KMS key
- Fix the copy action configuration in the backup plan or retry on-demand

SHOULD:
- Use customer-managed KMS keys in both regions with explicit cross-region grants
- Configure copy action lifecycle rules for the destination recovery point
- Test cross-region copy with an on-demand backup before relying on scheduled copies

MAY:
- Set up monitoring for cross-region copy job failures
- Implement automated alerting for failed copy jobs
- Document DR procedures including cross-region restore steps

## Common Issues

- symptoms: "KMS key error during cross-region copy"
  diagnosis: "The destination vault's KMS key does not allow the backup role from the source region."
  resolution: "Update the destination KMS key policy to grant kms:CreateGrant, kms:Decrypt, kms:GenerateDataKey to the backup role."

- symptoms: "Destination vault not found"
  diagnosis: "No backup vault exists in the destination region, or the vault name is incorrect."
  resolution: "Create a backup vault in the destination region with appropriate KMS encryption."

- symptoms: "Copy job stuck in RUNNING state"
  diagnosis: "Large recovery points take time to copy cross-region. Network or service issues may cause delays."
  resolution: "Monitor the copy job progress. Check AWS Health Dashboard for service issues in either region."

- symptoms: "Copy action in backup plan not triggering"
  diagnosis: "The copy action is misconfigured or the destination vault ARN is incorrect."
  resolution: "Verify the CopyActions in the backup plan rule with correct DestinationBackupVaultArn."

## Output Format

```yaml
root_cause: "cross_region_copy — <specific_cause>"
evidence:
  - type: copy_job
    content: "<copy job details and StatusMessage>"
  - type: destination_vault
    content: "<destination vault and KMS configuration>"
  - type: iam_role
    content: "<role permissions for cross-region operations>"
severity: HIGH
mitigation:
  immediate: "Fix KMS/IAM permissions and retry the copy"
  long_term: "Implement automated cross-region copy monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure destination vault exists with KMS encryption | YELLOW | New infrastructure in another region |
| Grant backup role permissions on destination KMS key | YELLOW | Cross-region IAM/KMS change |
| Fix copy action configuration | YELLOW | Changes backup plan copy behavior |
| Test cross-region copy with on-demand backup | GREEN | Testing — non-destructive |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Cross-region KMS key policy changes affect encryption access
- Destination vault configuration affects DR readiness

## Data Sensitivity

- Cross-region copy replicates backup data to another region — verify data residency requirements
- KMS key policies in destination region control encryption of replicated data
- Copy job details reveal DR strategy and data replication patterns

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER copy backup data to regions that violate data residency requirements
- NEVER use unencrypted vaults as copy destinations

## Phase 3 — Rollback

- If destination vault was created incorrectly: delete vault (only if empty) or update configuration
- If KMS key policy was changed: restore original key policy in destination region
- If copy action was misconfigured: update backup plan with correct copy action
- Document all cross-region changes for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
