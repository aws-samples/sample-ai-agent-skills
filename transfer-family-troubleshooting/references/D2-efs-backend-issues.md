---
title: "D2 — EFS Backend Issues"
description: "Diagnose EFS storage backend access, POSIX permissions, and mount issues"
status: active
severity: HIGH
triggers:
  - "EFS access denied"
  - "EFS permission"
  - "POSIX"
  - "Uid Gid"
  - "EFS mount"
  - "EFS backend error"
owner: devops-agent
objective: "Resolve EFS backend access and configuration issues for Transfer Family users"
context: "Transfer Family accesses EFS using the IAM role for mounting and POSIX user/group IDs for file-level permissions. Each user must have a PosixProfile with Uid and Gid. The EFS file system must have a mount target in the same VPC and subnets as the Transfer Family server. EFS access points can provide additional access control."
---

## Phase 1 — Triage

MUST:
- Check user's POSIX profile: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.PosixProfile'`
- Verify EFS file system exists: `aws efs describe-file-systems --file-system-id <fs-id>`
- Check mount targets in the server's VPC: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Verify the IAM role has EFS permissions: `aws iam list-attached-role-policies --role-name <role-name>`
- Check EFS file system policy: `aws efs describe-file-system-policy --file-system-id <fs-id>`

SHOULD:
- Verify mount target security groups allow NFS (port 2049) from the Transfer Family server's subnets
- Check if EFS access points are configured and match the user's POSIX profile
- Verify the home directory path exists on the EFS file system
- Check EFS throughput mode and performance: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[0].{ThroughputMode:ThroughputMode,PerformanceMode:PerformanceMode}'`

MAY:
- Check EFS lifecycle management settings
- Verify no EFS backup or replication is causing lock contention
- Check CloudWatch EFS metrics for throttling

## Phase 2 — Remediate

MUST:
- Set PosixProfile with valid Uid and Gid on each user (Uid/Gid 0 is root — avoid unless necessary)
- Ensure the IAM role has elasticfilesystem:ClientMount and elasticfilesystem:ClientWrite permissions
- Ensure mount target security groups allow inbound NFS (port 2049) from Transfer Family subnets
- Create the home directory path on EFS with correct POSIX ownership

SHOULD:
- Use secondary Gids in PosixProfile for shared directory access between users
- Set appropriate POSIX permissions on directories (e.g., 755 for home dirs, 770 for shared dirs)
- Use EFS access points to enforce root directory and POSIX identity

MAY:
- Enable EFS encryption in transit for additional security
- Configure provisioned throughput if burst credits are insufficient

## Common Issues

- symptoms: "Permission denied when creating files on EFS"
  diagnosis: "User's Uid/Gid does not have write permission on the target directory."
  resolution: "Change directory ownership or permissions to match the user's PosixProfile Uid/Gid."

- symptoms: "User cannot see files uploaded by another user"
  diagnosis: "Files created with one user's Uid/Gid are not readable by another user's Uid/Gid."
  resolution: "Use shared Gid via SecondaryGids and set group-readable permissions (e.g., 750)."

- symptoms: "EFS mount fails — Transfer Family server cannot access file system"
  diagnosis: "No mount target in the server's VPC/subnets, or mount target security group blocks NFS."
  resolution: "Create mount targets in the same subnets. Allow port 2049 inbound in mount target security groups."

## Output Format

```yaml
root_cause: "efs_backend — <specific_cause>"
evidence:
  - type: posix_profile
    content: "<user Uid/Gid configuration>"
  - type: mount_targets
    content: "<mount target locations and security groups>"
  - type: efs_policy
    content: "<file system policy>"
severity: HIGH
mitigation:
  immediate: "Fix POSIX permissions or mount target configuration"
  long_term: "Implement consistent POSIX identity management and EFS access points"
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

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
