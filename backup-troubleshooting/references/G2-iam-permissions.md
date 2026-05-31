---
title: "G2 — IAM Permission Issues"
description: "Diagnose IAM role and policy issues affecting AWS Backup operations"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "permission denied"
  - "IAM role"
  - "backup role"
  - "unauthorized"
  - "cannot assume role"
owner: devops-agent
objective: "Identify and resolve IAM permission issues preventing backup and restore operations"
context: "AWS Backup requires an IAM service role to perform operations. The role must trust backup.amazonaws.com and have permissions for each AWS service being backed up. The default role (AWSBackupDefaultServiceRole) uses AWS managed policies. Custom roles need explicit service permissions. Cross-account operations require roles in both accounts."
---

## Phase 1 — Triage

MUST:
- Check the backup/restore job error for permission-specific messages
- Get the IAM role details: `aws iam get-role --role-name <role-name>`
- Check the role's trust policy allows backup.amazonaws.com
- List attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- List inline policies: `aws iam list-role-policies --role-name <role-name>`

SHOULD:
- Check if the role has the AWS managed policies: AWSBackupServiceRolePolicyForBackup and AWSBackupServiceRolePolicyForRestores
- Verify service-specific permissions for the failing resource type
- Check for permission boundaries that may restrict the role
- Verify no SCP is blocking the backup operations

MAY:
- Use IAM Access Analyzer to evaluate the role's effective permissions
- Check CloudTrail for specific AccessDenied events
- Verify the role session duration is sufficient for long-running backup jobs

## Phase 2 — Remediate

MUST:
- Ensure the role trust policy includes: `{"Service": "backup.amazonaws.com"}` as a trusted principal
- Attach AWSBackupServiceRolePolicyForBackup for backup operations
- Attach AWSBackupServiceRolePolicyForRestores for restore operations
- Add service-specific permissions if using a custom role

SHOULD:
- Use the default AWSBackupDefaultServiceRole when possible
- For custom roles, follow least-privilege by only granting permissions for backed-up service types
- Document custom role permissions and their purpose

MAY:
- Create separate roles for backup and restore operations
- Implement role-based access control for different backup administrators
- Use AWS Organizations SCPs to enforce backup role standards

## Common Issues

- symptoms: "Cannot assume role — trust policy does not allow backup.amazonaws.com"
  diagnosis: "The IAM role's trust policy does not include backup.amazonaws.com as a trusted service."
  resolution: "Update the trust policy to allow sts:AssumeRole by backup.amazonaws.com."

- symptoms: "Backup succeeds for EC2 but fails for RDS"
  diagnosis: "The role has EC2 permissions but lacks RDS-specific permissions."
  resolution: "Add rds:CreateDBSnapshot, rds:DescribeDBInstances, rds:AddTagsToResource to the role."

- symptoms: "Restore fails with ec2:RunInstances denied"
  diagnosis: "The role has backup permissions but lacks restore-specific permissions."
  resolution: "Attach AWSBackupServiceRolePolicyForRestores or add ec2:RunInstances and related permissions."

- symptoms: "Permission boundary blocks backup operations"
  diagnosis: "The role has a permission boundary that does not include backup-related actions."
  resolution: "Update the permission boundary to include required backup and service-specific actions."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update role trust policy for backup.amazonaws.com | YELLOW | Changes who can assume the role |
| Attach AWSBackupServiceRolePolicyForBackup | GREEN | Standard AWS managed policy |
| Attach AWSBackupServiceRolePolicyForRestores | GREEN | Standard AWS managed policy |
| Add service-specific permissions to custom role | YELLOW | Expands role permissions — review scope |
| Update permission boundary | YELLOW | Changes effective permissions ceiling |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- IAM role is shared across multiple services beyond backup
- Permission boundary is managed by a security team with change control
- SCP blocking backup operations requires organizational policy change

## Data Sensitivity

- **Classification: HIGH**
- IAM role policies reveal backup infrastructure access patterns
- Trust policies expose which services and accounts can perform backup operations
- Permission boundaries reveal organizational security constraints
- CloudTrail AccessDenied events may contain sensitive resource ARNs

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest granting wildcard (*) permissions to backup roles
- **NEVER** suggest removing permission boundaries without security team approval

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Updated role trust policy | Revert trust policy to previous version |
| Attached managed backup policy | Detach the policy if it was not previously attached |
| Added service-specific permissions | Remove the added inline or managed policy |
| Updated permission boundary | Revert permission boundary to previous version |

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: iam_role
    content: "<role trust policy and attached policies>"
  - type: effective_permissions
    content: "<effective permissions analysis>"
  - type: error
    content: "<specific access denied error>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role permissions and trust policy"
  long_term: "Standardize backup roles with appropriate managed policies"
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
