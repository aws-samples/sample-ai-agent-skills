---
title: "A1 — Lustre Creation Failures"
description: "Diagnose why FSx for Lustre file system creation is failing"
status: active
severity: HIGH
triggers:
  - "lustre creation failed"
  - "lustre file system error"
  - "cannot create lustre"
  - "lustre FAILED status"
  - "lustre misconfigured"
owner: devops-agent
objective: "Identify the root cause of Lustre file system creation failures and resolve configuration issues"
context: "FSx for Lustre creation can fail due to subnet configuration, security group rules, IAM permissions, KMS key issues, invalid deployment type parameters, or S3 data repository configuration. The file system status message and CloudTrail events provide the most diagnostic value."
---

## Phase 1 — Triage

MUST:
- Describe the file system to check status: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check the Lifecycle and FailureDetails fields for error messages
- Verify the subnet exists and has available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-id>`
- Check security group allows Lustre traffic (port 988): `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`
- Verify KMS key is valid and accessible: `aws kms describe-key --key-id <key-id>`
- Confirm the IAM service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForAmazonFSx`
- Verify deployment type parameters (scratch vs persistent, storage capacity increments)

MAY:
- Check VPC route tables for connectivity: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`
- Verify S3 bucket accessibility if DRA is configured: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Check for service quotas: `aws service-quotas get-service-quota --service-code fsx --quota-code L-A67D2BBF`

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (subnet, security group, KMS, IAM)
- Ensure security group allows inbound/outbound TCP port 988 within the security group
- Verify storage capacity meets minimum requirements (1.2 TiB for scratch, 1.2 TiB for persistent)

SHOULD:
- Use persistent deployment type for production workloads requiring durability
- Configure automatic backups for persistent file systems
- Tag file systems for cost tracking and management

MAY:
- Set up CloudWatch alarms for file system health monitoring
- Configure SNS notifications for file system events

## Common Issues

- symptoms: "File system status FAILED with subnet error"
  diagnosis: "The subnet does not have enough available IP addresses or is in an unsupported AZ."
  resolution: "Use a subnet with available IPs. Lustre requires at least one IP per file server."

- symptoms: "Security group error during creation"
  diagnosis: "The security group does not allow Lustre network traffic on port 988."
  resolution: "Add inbound rule allowing TCP 988 from the security group itself (self-referencing rule)."

- symptoms: "KMS key access denied"
  diagnosis: "The FSx service-linked role cannot access the specified KMS key."
  resolution: "Update KMS key policy to grant fsx.amazonaws.com kms:CreateGrant, kms:Decrypt, kms:GenerateDataKey."

- symptoms: "Invalid storage capacity for deployment type"
  diagnosis: "Storage capacity does not meet the minimum or increment requirements for the deployment type."
  resolution: "Scratch: multiples of 1.2/2.4/4.8 TiB. Persistent SSD: multiples of 1.2 TiB. Persistent HDD: multiples of 6 TiB."

## Output Format

```yaml
root_cause: "lustre_creation_failure — <specific_cause>"
evidence:
  - type: file_system
    content: "<describe-file-systems output with FailureDetails>"
  - type: subnet
    content: "<subnet availability and configuration>"
  - type: security_group
    content: "<security group rules analysis>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration issue and recreate the file system"
  long_term: "Create validated CloudFormation templates for consistent Lustre deployments"
```


---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — FSx provides shared file storage for compute workloads, databases, and analytics |
| Data Risk | CRITICAL — File systems contain production data, application state, and potentially irreplaceable datasets |
| Recovery Complexity | HIGH — File system recreation requires backup restoration and client remounting across all consumers |
| Blast Radius | HIGH — File system outages affect all connected compute instances and dependent applications |

## Escalation Conditions

- File system enters FAILED or MISCONFIGURED state with no clear remediation path
- Multi-AZ failover not completing or standby file server unavailable during AZ event
- Data corruption detected on file system volumes or inconsistent backup state
- KMS key disabled or deleted causing file system inaccessibility
- Storage capacity at 100% utilization with no ability to scale (quota limits reached)
- Backup or restore operations failing on business-critical file systems

## Data Sensitivity

- File systems may contain regulated data (financial records, healthcare data, PII)
- Backup contents mirror production data sensitivity classifications
- Active Directory integration credentials reveal domain architecture and trust relationships
- NFS export policies and NTFS ACLs expose organizational access control structure
- ONTAP SVM configurations reveal multi-tenant data isolation boundaries

## Prohibited Actions

- NEVER suggest deleting file systems without confirming backups exist and are restorable
- NEVER suggest disabling encryption or using unencrypted file systems for any workload
- NEVER suggest reducing storage capacity — this operation is irreversible on all FSx types
- NEVER modify Active Directory integration on production Windows FSx without maintenance window
- NEVER remove or modify KMS key policies without verifying all dependent file systems
- NEVER change security group rules that could disconnect active file system clients

## Phase 3 — Rollback

- If file system configuration was changed: revert settings and verify client connectivity is restored
- If storage scaling failed: wait for pending operations to complete, verify file system returns to AVAILABLE
- If backup was deleted: check AWS Backup vault for retained copies, recreate from most recent available backup
- If security group was modified: restore previous rules immediately and verify all client mounts are functional
- If KMS key was affected: re-enable key or cancel deletion, verify file system recovers automatically
- Document all rollback actions with file system ID, affected clients, and data integrity verification steps

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest removing AD integration without review"
  - "NEVER suggest public subnet placement"
