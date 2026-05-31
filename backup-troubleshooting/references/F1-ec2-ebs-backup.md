---
title: "F1 — EC2/EBS Backup Issues"
description: "Diagnose EC2 instance and EBS volume backup failures specific to compute resources"
status: active
severity: HIGH
triggers:
  - "EC2 backup failed"
  - "EBS snapshot failed"
  - "AMI backup"
  - "instance backup"
  - "volume backup"
owner: devops-agent
objective: "Identify and resolve EC2/EBS-specific backup issues"
context: "EC2 backups create AMIs (which include EBS snapshots). EBS backups create snapshots directly. EC2 backups can include or exclude specific volumes. First EBS snapshots are full copies; subsequent ones are incremental. Instance state, volume encryption, and IAM permissions are common failure points."
---

## Phase 1 — Triage

MUST:
- Describe the failed backup job: `aws backup describe-backup-job --backup-job-id <job-id>`
- Check the resource type (EC2 or EBS) and resource ARN
- Verify the instance/volume state: `aws ec2 describe-instances --instance-ids <id>` or `aws ec2 describe-volumes --volume-ids <id>`
- Check the backup role has EC2 permissions: `aws iam get-role-policy --role-name <role> --policy-name <policy>`
- Verify KMS key permissions if the volume is encrypted

SHOULD:
- Check if the EC2 instance has the required tags for backup selection
- Verify no pending snapshots are blocking new ones: `aws ec2 describe-snapshots --filters Name=status,Values=pending --owner-ids self`
- Check EBS snapshot limits: `aws service-quotas get-service-quota --service-code ebs --quota-code L-309BACF6`
- Check if the instance is in a transitional state (stopping, pending, shutting-down)

MAY:
- Check if application-consistent backup is configured (requires SSM agent and VSS)
- Verify Windows VSS backup configuration for Windows instances
- Check if the instance has instance store volumes (not backed up by AWS Backup)

## Phase 2 — Remediate

MUST:
- Ensure the backup role has ec2:CreateSnapshot, ec2:CreateImage, ec2:DescribeInstances, ec2:DescribeVolumes permissions
- Grant KMS permissions if volumes use customer-managed encryption keys
- Ensure the instance/volume is in a valid state before retrying

SHOULD:
- For Windows instances, install and configure the SSM agent for VSS-based application-consistent backups
- Exclude unnecessary volumes from EC2 backups to reduce time and cost
- Use EBS fast snapshot restore for frequently backed-up volumes

MAY:
- Enable EBS snapshot archiving for long-term retention cost savings
- Implement pre/post backup scripts via SSM for application consistency
- Monitor snapshot creation times for capacity planning

## Common Issues

- symptoms: "EC2 backup fails with permission error"
  diagnosis: "The backup role lacks ec2:CreateImage or ec2:CreateSnapshot permissions."
  resolution: "Attach AWSBackupServiceRolePolicyForBackup or add specific EC2 permissions."

- symptoms: "EBS snapshot takes very long or times out"
  diagnosis: "First snapshot of a large volume is a full copy. Subsequent snapshots are incremental."
  resolution: "Allow extra time for the first snapshot. Increase the completion window."

- symptoms: "Windows EC2 backup is not application-consistent"
  diagnosis: "VSS (Volume Shadow Copy Service) is not configured or SSM agent is not running."
  resolution: "Install SSM agent, enable advanced backup features, and configure Windows VSS."

- symptoms: "Instance store volumes not included in backup"
  diagnosis: "AWS Backup only backs up EBS volumes. Instance store volumes are ephemeral."
  resolution: "Instance store data must be backed up separately. Use EBS volumes for persistent data."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Attach AWSBackupServiceRolePolicyForBackup to role | GREEN | Grants standard backup permissions |
| Grant KMS permissions for encrypted volumes | YELLOW | Expands key access — review scope |
| Enable EBS fast snapshot restore | YELLOW | Incurs additional cost per AZ |
| Exclude unnecessary volumes from EC2 backup | YELLOW | Changes backup scope — verify no critical data excluded |
| Install SSM agent for VSS backups | GREEN | Non-destructive agent installation |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- EC2 instance runs production workloads and backup window is critical
- Volume encryption key is shared across multiple services
- Backup failure affects compliance reporting or audit requirements

## Data Sensitivity

- **Classification: HIGH**
- EBS snapshots contain full volume data including application state
- EC2 AMI backups include OS configuration and potentially embedded credentials
- Instance metadata reveals infrastructure architecture and sizing
- KMS key configurations reveal encryption strategy per resource type

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest terminating an EC2 instance to resolve a backup issue
- **NEVER** suggest disabling volume encryption to work around KMS permission errors

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Attached backup policy to IAM role | Detach the policy from the role if it was not previously attached |
| Granted KMS key permissions | Remove the added KMS key policy statement |
| Enabled EBS fast snapshot restore | Disable fast snapshot restore to stop incurring charges |
| Excluded volumes from EC2 backup | Re-add excluded volumes to the backup selection |

## Output Format

```yaml
root_cause: "ec2_ebs_backup — <specific_cause>"
evidence:
  - type: backup_job
    content: "<job details and error>"
  - type: instance_state
    content: "<EC2/EBS resource state>"
  - type: permissions
    content: "<IAM role permissions for EC2/EBS>"
severity: HIGH
mitigation:
  immediate: "Fix permissions/state and retry backup"
  long_term: "Configure application-consistent backups and monitoring"
```

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
