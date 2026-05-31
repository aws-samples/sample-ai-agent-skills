---
title: "H1 — IAM Roles"
description: "Diagnose IAM role and permission issues for DMS"
status: active
severity: HIGH
triggers:
  - "IAM error"
  - "access denied DMS"
  - "role not found"
  - "dms-vpc-role"
  - "permission denied"
owner: devops-agent
objective: "Fix IAM role configuration for DMS replication instances and tasks"
context: "DMS requires specific IAM roles: dms-vpc-role for VPC access, dms-cloudwatch-logs-role for logging, and dms-access-for-endpoint for S3/Redshift/DynamoDB endpoints. Missing or misconfigured roles cause creation failures and access errors."
---

## Phase 1 — Triage

MUST:
- Check if dms-vpc-role exists: `aws iam get-role --role-name dms-vpc-role`
- Check if dms-cloudwatch-logs-role exists: `aws iam get-role --role-name dms-cloudwatch-logs-role`
- Check role trust policies allow DMS service: `aws iam get-role --role-name dms-vpc-role --query 'Role.AssumeRolePolicyDocument'`
- Check attached policies: `aws iam list-attached-role-policies --role-name dms-vpc-role`

SHOULD:
- Verify the DMS service principal (dms.amazonaws.com) is in trust policy
- Check for S3/Redshift/DynamoDB endpoint roles if using those targets
- Review CloudTrail for access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=dms.amazonaws.com --max-results 20`

MAY:
- Check if roles were recently modified or deleted
- Verify KMS key policies if using encrypted endpoints

## Phase 2 — Remediate

MUST:
- Create dms-vpc-role with AmazonDMSVPCManagementRole policy if missing
- Create dms-cloudwatch-logs-role with AmazonDMSCloudWatchLogsRole policy if missing
- Ensure trust policy includes: `{"Service": "dms.amazonaws.com"}`

SHOULD:
- For S3 endpoints, create role with S3 read/write permissions
- For Redshift endpoints, ensure role has Redshift access
- Test by creating a simple replication instance after fixing roles

MAY:
- Use least-privilege policies instead of AWS managed policies
- Set up IAM Access Analyzer to monitor DMS role usage

## Common Issues

- symptoms: "Replication instance creation fails with IAM error"
  diagnosis: "dms-vpc-role does not exist or lacks required policy."
  resolution: "Create dms-vpc-role with AmazonDMSVPCManagementRole managed policy."

- symptoms: "Cannot enable CloudWatch logging"
  diagnosis: "dms-cloudwatch-logs-role missing or misconfigured."
  resolution: "Create role with AmazonDMSCloudWatchLogsRole policy and DMS trust."

- symptoms: "S3 target endpoint access denied"
  diagnosis: "Endpoint service access role lacks S3 permissions."
  resolution: "Create/update the service access role with S3 bucket read/write permissions."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create dms-vpc-role with managed policy | GREEN | Standard role creation — non-destructive |
| Create dms-cloudwatch-logs-role | GREEN | Standard role creation — non-destructive |
| Update trust policy for DMS service | YELLOW | Changes who can assume the role |
| Create S3 endpoint service access role | YELLOW | Grants S3 access — review scope |
| Use least-privilege policies | GREEN | Security best practice |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- IAM role changes affect other DMS tasks or services sharing the roles
- Role creation requires organizational IAM approval process

## Data Sensitivity

- **Classification: HIGH**
- IAM role policies reveal DMS infrastructure access patterns
- Trust policies expose which services can perform replication operations
- S3/Redshift endpoint roles reveal target data store access
- CloudTrail AccessDenied events may contain sensitive resource ARNs

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest granting wildcard (*) permissions to DMS roles
- **NEVER** suggest deleting IAM roles while replication tasks are running

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Created dms-vpc-role | Delete the role if it was created incorrectly |
| Created dms-cloudwatch-logs-role | Delete the role if it was created incorrectly |
| Updated trust policy | Revert trust policy to previous version |
| Created S3 endpoint role | Delete the role if no longer needed |

## Output Format

```yaml
root_cause: "iam_role — <specific_cause>"
evidence:
  - type: role_status
    content: "<role existence and configuration>"
  - type: trust_policy
    content: "<trust policy document>"
  - type: attached_policies
    content: "<attached policy list>"
severity: HIGH
mitigation:
  immediate: "Create or fix the required IAM roles"
  long_term: "Automate DMS IAM role creation with IaC"
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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
