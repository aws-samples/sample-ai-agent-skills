---
title: "C2 â€” EFS IAM Authorization Issues"
description: "Diagnose IAM-based access control failures for EFS"
status: active
severity: MEDIUM
triggers:
  - "IAM authorization"
  - "access denied EFS"
  - "file system policy"
  - "elasticfilesystem:ClientMount"
owner: devops-agent
objective: "Identify and fix IAM authorization issues for EFS access"
context: "EFS supports IAM-based access control via file system policies and IAM policies. When IAM authorization is enabled, clients must use the EFS mount helper with -o iam option. The instance profile or task role must have elasticfilesystem:Client* permissions."
---

## Phase 1 â€” Triage

MUST:
- Check file system policy: `aws efs describe-file-system-policy --file-system-id <fs-id>`
- Check if the policy requires IAM authorization
- Verify the instance/task role has EFS permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names elasticfilesystem:ClientMount elasticfilesystem:ClientWrite --resource-arns arn:aws:elasticfilesystem:<region>:<account>:file-system/<fs-id>`

SHOULD:
- Verify the mount command uses -o iam option
- Check if the instance has an instance profile attached
- Verify the EFS mount helper is installed and configured

MAY:
- Check CloudTrail for EFS access denied events
- Verify the file system policy conditions (IP, VPC, etc.)

## Phase 2 â€” Remediate

MUST:
- Add elasticfilesystem:ClientMount and ClientWrite permissions to the IAM role
- Mount with IAM: `sudo mount -t efs -o iam <fs-id>:/ /mnt/efs`
- Fix file system policy if it's overly restrictive

SHOULD:
- Use access points with IAM for fine-grained access control
- Ensure the mount helper is up to date

MAY:
- Implement least-privilege file system policies

## Common Issues

- symptoms: "Access denied when mounting with IAM"
  diagnosis: "IAM role lacks elasticfilesystem:ClientMount permission."
  resolution: "Add the required EFS permissions to the IAM role."

- symptoms: "Mount works without IAM but fails with -o iam"
  diagnosis: "File system policy requires IAM but the role doesn't have permissions."
  resolution: "Add EFS permissions to the role or adjust the file system policy."

- symptoms: "Read-only access when expecting read-write"
  diagnosis: "IAM role has ClientMount but not ClientWrite permission."
  resolution: "Add elasticfilesystem:ClientWrite to the IAM policy."

## Output Format

```yaml
root_cause: "iam_auth_failure â€” <specific_cause>"
evidence:
  - type: file_system_policy
    content: "<EFS resource policy>"
  - type: iam_simulation
    content: "<IAM simulation results>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions or file system policy"
  long_term: "Implement IAM-based access control with least privilege"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — EFS provides shared storage for multiple compute instances |
| Data Risk | CRITICAL — File system contains application data and shared state |
| Recovery Complexity | HIGH — Data loss from file system deletion is unrecoverable without backups |
| Blast Radius | HIGH — Mount or performance issues affect all connected instances |

## Escalation Conditions

- File system unavailable or mount failures affecting multiple production instances
- Throughput degradation causing application timeouts across the fleet
- Burst credits depleted with no provisioned throughput configured
- Security group changes blocking NFS access for running workloads
- Replication lag exceeding RPO thresholds for disaster recovery
- IAM authorization failures preventing application access to file system

## Data Sensitivity

- File system contents may include application data, configuration files, and shared secrets
- Access point configurations reveal application directory structure and POSIX permissions
- Mount target IPs and DNS names are internal infrastructure identifiers
- Encryption keys (KMS) protect data at rest — key access must be tightly controlled
- EFS access logs may contain file paths revealing business logic or data organization

## PROHIBITED Actions

- NEVER suggest deleting a file system with active mount targets
- NEVER remove mount targets from availability zones with running instances
- NEVER change security group rules without verifying all connected instance groups
- NEVER disable encryption in transit on file systems handling sensitive data
- NEVER modify access point POSIX permissions without understanding application requirements
- NEVER switch throughput modes during peak usage without capacity planning

## Phase 3 — Rollback

- If mount target was removed: recreate in the same subnet with the same security groups
- If security group rules were changed: restore previous inbound TCP 2049 rules from backup
- If throughput mode was changed: revert to previous mode and monitor burst credit balance
- If access point permissions were modified: restore original POSIX UID/GID and directory path
- If lifecycle policy was changed: revert transition rules and verify IA storage class files
- Document all rollback actions with file system ID, mount target IDs, and timestamps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing IAM authorization"
  - "NEVER suggest overly broad file system policies"
