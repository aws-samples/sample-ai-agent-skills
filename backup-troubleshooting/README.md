# AWS Backup Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Backup problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Backup when the console alone isn't enough — backup job failure analysis, restore operation debugging, backup plan configuration issues, vault management and lock problems, cross-account and cross-region operations, KMS encryption errors, IAM permission issues, service-specific backup problems, and compliance auditing.

### Activate When

- Backup job failures or errors
- Backup job timeouts
- Partial backup failures
- On-demand backup errors
- Restore operation failures
- Cross-account restore issues
- Point-in-time restore problems
- Backup plan misconfiguration
- Schedule not triggering as expected
- Resource selection not matching expected resources
- Vault access policy denials
- Vault lock configuration issues
- Cross-region copy failures
- Organization backup setup problems
- Cross-account copy failures
- EC2/EBS backup issues
- RDS backup or snapshot problems
- DynamoDB backup failures
- KMS encryption errors during backup or restore
- IAM permission errors for backup operations
- Audit Manager framework non-compliance
- Compliance reporting gaps or failures

---

## Skill Structure

```
backup-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-backup-job-failures.md
    ├── A2-backup-job-timeout.md
    ├── A3-partial-backup-failures.md
    ├── A4-on-demand-backup-errors.md
    ├── B1-restore-failures.md
    ├── B2-cross-account-restore.md
    ├── B3-point-in-time-restore.md
    ├── C1-plan-configuration.md
    ├── C2-schedule-issues.md
    ├── C3-resource-selection.md
    ├── D1-vault-access-policy.md
    ├── D2-vault-lock.md
    ├── D3-cross-region-copy.md
    ├── E1-organization-backup.md
    ├── E2-cross-account-copy.md
    ├── F1-ec2-ebs-backup.md
    ├── F2-rds-backup.md
    ├── F3-dynamodb-backup.md
    ├── G1-kms-encryption.md
    ├── G2-iam-permissions.md
    ├── H1-audit-manager.md
    ├── H2-compliance-reporting.md
    ├── Z1-general-troubleshooting.md
    ├── backup-guardrails.md
    └── backup-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Backup Jobs** | A1–A4 | Job failures, timeout, partial failures, on-demand backup errors |
| **B — Restore** | B1–B3 | Restore failures, cross-account restore, point-in-time restore |
| **C — Backup Plans** | C1–C3 | Plan configuration, schedule issues, resource selection |
| **D — Vault** | D1–D3 | Vault access policy, vault lock, cross-region copy |
| **E — Cross-Account** | E1–E2 | Organization backup, cross-account copy |
| **F — Service-Specific** | F1–F3 | EC2/EBS backup, RDS backup, DynamoDB backup |
| **G — Security** | G1–G2 | KMS encryption, IAM permissions |
| **H — Compliance** | H1–H2 | Audit manager, compliance reporting |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/backup-guardrails.md` covering vault lock irreversibility, cross-account organization requirement, cold storage transition minimums, lifecycle rule interactions, resource selection behavior, supported services list, backup vault encryption, IAM role requirements, continuous vs snapshot backups, restore testing limitations, backup plan rule limits, and recovery point retention.

---

## Investigation Workflow

1. **Triage** — List failed backup/restore jobs, check CloudWatch metrics, describe specific job failures
2. **Deep Dive** — Examine backup plans, selections, vault policies, recovery points
3. **Detailed** — Inspect copy jobs, KMS keys, IAM roles, CloudTrail events, Audit Manager frameworks

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `backup:*`, `ec2:Describe*`, `rds:Describe*`, `dynamodb:Describe*`, `kms:DescribeKey`, `kms:ListGrants`, `iam:GetRole`, `iam:GetRolePolicy`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `organizations:Describe*`, `tag:GetResources`
- CloudWatch metrics enabled for AWS Backup
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# List recent failed backup jobs
aws backup list-backup-jobs --by-state FAILED --max-results 10

# Describe a specific failed job
aws backup describe-backup-job --backup-job-id <job-id>

# Check backup plan configuration
aws backup get-backup-plan --backup-plan-id <plan-id>

# List recovery points in a vault
aws backup list-recovery-points-by-backup-vault --backup-vault-name <vault-name>

# Check vault access policy
aws backup get-backup-vault-access-policy --backup-vault-name <vault-name>

# List failed restore jobs
aws backup list-restore-jobs --by-status FAILED --max-results 10
```

---

## License

MIT-0
