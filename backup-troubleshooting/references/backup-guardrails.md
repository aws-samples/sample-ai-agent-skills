# AWS Backup Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Backup issue.

## Guardrail 1: Vault Lock in Compliance Mode Is IRREVERSIBLE
Once a backup vault lock is applied in compliance mode, it CANNOT be removed — not by any user, not by the root account, not by AWS Support. The vault's retention policies become immutable. Governance mode CAN be removed by users with `backup:DeleteBackupVaultLockConfiguration` permission. Always verify the lock mode before applying. A cooling-off period (3–36 days) exists before compliance mode becomes permanent — changes can be made during this period.

## Guardrail 2: Cross-Account Backup Requires AWS Organizations
Cross-account backup is only available for accounts within the same AWS Organization. The management account must enable cross-account backup and cross-account monitoring in AWS Backup settings. Standalone accounts cannot use cross-account backup features. The destination account's vault must have an access policy granting the source account permissions.

## Guardrail 3: Cold Storage Transition Has a 90-Day Minimum
When configuring lifecycle rules, the transition to cold storage requires a minimum retention period of 90 days. The delete-after value must be at least 90 days greater than the move-to-cold-after value. Not all resource types support cold storage — only EBS snapshots, EFS backups, and DynamoDB backups support cold storage transition. EC2 AMIs, RDS snapshots, and S3 backups do NOT support cold storage.

## Guardrail 4: Lifecycle Rules — Cold and Delete Interact
The MoveToColdStorageAfterDays and DeleteAfterDays values interact. If DeleteAfterDays is set, it must be at least 90 days greater than MoveToColdStorageAfterDays. If only DeleteAfterDays is set without cold storage transition, there is no minimum. Setting MoveToColdStorageAfterDays without DeleteAfterDays means the recovery point is kept in cold storage indefinitely. Never assume lifecycle values are independent.

## Guardrail 5: Tag-Based Selection Is Dynamic, ARN-Based Is Static
Tag-based resource selection automatically includes new resources that match the tag conditions. ARN-based selection only includes the specific resources listed. If a resource matches both a tag-based and ARN-based selection in different backup plans, it may be backed up multiple times. Tag conditions support StringEquals and StringLike operators only — no regex or complex matching.

## Guardrail 6: Not All AWS Services Support AWS Backup
Supported services: EC2, EBS, RDS, Aurora, DynamoDB, EFS, FSx (all types), Storage Gateway, DocumentDB, Neptune, S3, VMware (via Backup Gateway), SAP HANA on EC2, CloudFormation, Redshift, and Timestream. Services NOT supported include: ElastiCache, OpenSearch, MSK, MemoryDB, Keyspaces, and QLDB. Always verify service support before creating backup plans. The supported services list is updated periodically.

## Guardrail 7: Backup Vault Encryption Is Mandatory and Immutable
Every backup vault is encrypted with a KMS key specified at creation time. The encryption key CANNOT be changed after vault creation. The default vault uses the AWS-managed key `aws/backup`. For cross-account copy, the destination vault's KMS key must grant `kms:Decrypt` and `kms:CreateGrant` to the source account's backup role. For cross-region copy, a KMS key in the destination region is required.

## Guardrail 8: IAM Role Is Required for All Backup Operations
AWS Backup requires an IAM role (service role) to perform backup and restore operations. The default role is `AWSBackupDefaultServiceRole`. The role must have permissions for the specific AWS services being backed up. For cross-account operations, roles must exist in both source and destination accounts. Missing or misconfigured roles are the most common cause of backup job failures.

## Guardrail 9: Continuous Backup vs Snapshot Backup Are Different
Continuous backup (point-in-time recovery) is only supported for S3, RDS, Aurora, and SAP HANA. It enables restore to any point within the retention period (up to 35 days for RDS/Aurora). Snapshot-based backup creates discrete recovery points at scheduled times. Do not confuse the two — continuous backup has different retention limits and supported services than snapshot backup.

## Guardrail 10: Restore Testing Creates Real Resources
Restore testing plans create actual AWS resources during validation. These resources are automatically cleaned up after the validation period, but they incur costs while running. Restore testing does NOT validate data integrity — it only confirms the restore operation completes successfully. Failed cleanup may leave orphaned resources that continue to incur charges.

## Guardrail 11: Backup Plan Rules Have Limits
A backup plan can have up to 10 rules. Each rule defines a schedule, lifecycle, vault, and optional copy action. Rules within the same plan can target different vaults. If multiple rules have overlapping schedules, the resource is backed up multiple times. Backup windows define when a job STARTS — the job may run beyond the window. The completion window is separate and defines when to cancel if not finished.

## Guardrail 12: Recovery Point Retention Cannot Be Shortened Below Vault Lock Minimum
If a vault lock is configured with a minimum retention period, no recovery point in that vault can have a lifecycle that deletes it before the minimum retention. Attempts to create backups with shorter retention will fail. This applies to both backup plan rules and on-demand backups. The vault lock minimum retention overrides any lifecycle configuration.
