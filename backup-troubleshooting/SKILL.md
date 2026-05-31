---
name: backup-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Backup problems
  by analyzing backup jobs, restore operations, backup plans, vault
  configurations, cross-account setups, and following structured runbooks.
  Activate when: backup job failures, restore failures, backup plan
  misconfiguration, vault access issues, vault lock problems, cross-region
  copy failures, cross-account backup issues, KMS encryption errors,
  IAM permission problems, service-specific backup issues (EC2, RDS,
  DynamoDB), compliance reporting gaps, audit manager findings, lifecycle
  rule confusion, resource selection failures, on-demand backup errors,
  point-in-time restore issues, or the user says something is wrong
  with AWS Backup without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with AWS Backup, EC2, RDS, DynamoDB,
  EBS, EFS, S3, KMS, IAM, CloudWatch, CloudTrail, Organizations, and
  optionally FSx, Storage Gateway, and Aurora permissions.
---

# AWS Backup Diagnostics

## When to use

Any AWS Backup investigation where the console alone is insufficient — backup job analysis, restore troubleshooting, plan configuration debugging, vault management, cross-account/cross-region operations, encryption issues, compliance auditing, or service-specific backup problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws backup list-backup-jobs --by-state FAILED --max-results 20
aws backup list-backup-jobs --by-state COMPLETED --max-results 10
aws backup describe-backup-job --backup-job-id <job-id>
aws backup list-restore-jobs --by-status FAILED --max-results 20
aws backup list-backup-vaults
aws backup describe-backup-vault --backup-vault-name <vault-name>
aws cloudwatch get-metric-statistics --namespace AWS/Backup --metric-name NumberOfBackupJobsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws backup get-backup-plan --backup-plan-id <plan-id>
aws backup list-backup-selections --backup-plan-id <plan-id>
aws backup get-backup-selection --backup-plan-id <plan-id> --selection-id <selection-id>
aws backup get-backup-vault-access-policy --backup-vault-name <vault-name>
aws backup get-backup-vault-notifications --backup-vault-name <vault-name>
aws backup list-recovery-points-by-backup-vault --backup-vault-name <vault-name>
aws backup describe-recovery-point --backup-vault-name <vault-name> --recovery-point-arn <rp-arn>
```

### Step 3 — Detailed investigation

```
aws backup list-copy-jobs --by-state FAILED
aws backup describe-copy-job --copy-job-id <copy-job-id>
aws backup list-backup-plan-versions --backup-plan-id <plan-id>
aws backup list-frameworks
aws backup describe-framework --framework-name <framework-name>
aws backup list-report-plans
aws backup describe-report-plan --report-plan-name <plan-name>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=backup.amazonaws.com --max-results 20
aws kms describe-key --key-id <key-id>
aws iam get-role --role-name <backup-role-name>
```

Read `references/backup-guardrails.md` before concluding on any AWS Backup issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-backup-jobs` | List backup jobs by state, resource, vault |
| `describe-backup-job` | Full details of a specific backup job |
| `list-restore-jobs` | List restore jobs by status |
| `describe-restore-job` | Full details of a specific restore job |
| `list-backup-vaults` | List all backup vaults |
| `describe-backup-vault` | Vault details including encryption and lock |
| `get-backup-plan` | Backup plan rules and schedule |
| `list-backup-selections` | Resources selected by a plan |
| `get-backup-vault-access-policy` | Vault resource policy |
| `list-recovery-points-by-backup-vault` | Recovery points in a vault |
| `describe-recovery-point` | Recovery point details and lifecycle |
| `list-copy-jobs` | Cross-region/cross-account copy jobs |
| `list-frameworks` | Backup Audit Manager frameworks |
| `describe-framework` | Framework controls and compliance |
| `list-report-plans` | Audit Manager report plans |

## Gotchas: AWS Backup

- Backup vault lock (compliance mode) is IRREVERSIBLE. Once a vault lock is applied in compliance mode, it CANNOT be removed — not even by the root account. The vault and its retention policies are immutable. Governance mode CAN be removed by users with sufficient permissions. Always confirm the mode before applying.
- Cross-account backup requires AWS Organizations. The management account must enable cross-account backup in the AWS Backup settings. Both source and destination accounts must be in the same organization. The destination vault must have an access policy allowing the source account.
- Lifecycle rules: transition to cold storage after X days, delete after Y days. Cold storage transition requires a minimum of 90 days retention. Not all resource types support cold storage (only EBS, EFS, and DynamoDB support it). The delete-after value must be at least 90 days greater than the transition-to-cold value.
- Backup plan vs on-demand backup: backup plans run on a schedule with defined lifecycle rules. On-demand backups use the default lifecycle or specified parameters. On-demand backups do NOT inherit backup plan lifecycle rules — they must be specified explicitly.
- Resource selection by tags vs ARN: tag-based selection is dynamic (new resources with matching tags are automatically included). ARN-based selection is static. If a resource is selected by both tag and ARN in different selections, it may be backed up twice. Tag conditions support StringEquals and StringLike operators.
- Not all AWS services are supported by AWS Backup. Supported services include: EC2, EBS, RDS, Aurora, DynamoDB, EFS, FSx, Storage Gateway, DocumentDB, Neptune, S3, VMware (via Backup Gateway), SAP HANA, CloudFormation, Redshift, and Timestream. Services like ElastiCache, OpenSearch, and MSK are NOT supported.
- Backup vault access policy controls who can access recovery points. Without a policy, only the vault creator account has access. For cross-account access, the vault policy must explicitly grant permissions to the source account. Vault policies use IAM policy syntax.
- Encryption with KMS: each backup vault is encrypted with a KMS key. The default vault uses an AWS-managed key (aws/backup). Custom vaults can use customer-managed KMS keys. Cross-account copy requires the destination vault's KMS key to allow the source account. Cross-region copy requires the destination region's KMS key.
- Backup Gateway for VMware: enables backup of VMware VMs to AWS. Requires a Backup Gateway appliance deployed on-premises. The gateway must have network connectivity to both vCenter and AWS. Hypervisor must be registered with AWS Backup.
- Restore testing: AWS Backup supports automated restore testing via restore testing plans. These validate that backups are restorable. Restore testing creates temporary resources that are automatically cleaned up. Use restore testing to meet compliance requirements for backup validation.
- Backup Audit Manager: provides frameworks with pre-built controls for backup compliance. Controls include backup frequency, minimum retention, vault lock, cross-region/cross-account backup. Frameworks generate daily compliance reports. Non-compliant resources are flagged with specific control failures.

## Anti-hallucination rules

1. Always cite specific backup job IDs, vault names, or API responses as evidence.
2. Vault lock in compliance mode is IRREVERSIBLE. Never suggest it can be removed or modified.
3. Cross-account backup requires Organizations. Never suggest it works with standalone accounts.
4. Cold storage transition has a 90-day minimum retention requirement. Never suggest shorter periods.
5. Not all services support AWS Backup. Always verify service support before recommending backup plans.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Backup Jobs | A1-A4 | Job failures, timeout, partial failures, on-demand backup errors |
| B — Restore | B1-B3 | Restore failures, cross-account restore, point-in-time restore |
| C — Backup Plans | C1-C3 | Plan configuration, schedule issues, resource selection |
| D — Vault | D1-D3 | Vault access policy, vault lock, cross-region copy |
| E — Cross-Account | E1-E2 | Organization backup, cross-account copy |
| F — Service-Specific | F1-F3 | EC2/EBS backup, RDS backup, DynamoDB backup |
| G — Security | G1-G2 | KMS encryption, IAM permissions |
| H — Compliance | H1-H2 | Audit manager, compliance reporting |
| Z — Catch-All | Z1 | General troubleshooting |
