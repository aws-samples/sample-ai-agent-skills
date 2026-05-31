---
title: "A1 â€” EFS Mount Failures"
description: "Diagnose EFS mount command failures"
status: active
severity: HIGH
triggers:
  - "mount failed"
  - "mount.nfs"
  - "Connection refused"
  - "No such file or directory"
  - "mount: /efs: bad option"
owner: devops-agent
objective: "Identify why the EFS mount command is failing and fix the configuration"
context: "EFS mount failures are caused by missing mount targets, security group blocks, missing amazon-efs-utils package, incorrect mount command syntax, or DNS resolution failures. The mount helper (amazon-efs-utils) is recommended over raw NFS mount."
---

## Phase 1 â€” Triage

MUST:
- Check mount targets exist in the instance's AZ: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Check mount target security groups: `aws efs describe-mount-target-security-groups --mount-target-id <mt-id>`
- Verify security group allows inbound TCP 2049: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[].IpPermissions[?ToPort==`2049`]'`
- Check if amazon-efs-utils is installed on the instance

SHOULD:
- Verify the instance's security group allows outbound to port 2049
- Check if the file system is in Available state: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].LifeCycleState'`
- Verify the mount target is in the same VPC as the instance

MAY:
- Check NFS client logs on the instance: `dmesg | grep nfs`
- Test network connectivity: `nc -zv <mount-target-ip> 2049`

## Phase 2 â€” Remediate

MUST:
- Create mount target in the instance's AZ if missing
- Fix security group rules to allow TCP 2049
- Install amazon-efs-utils: `sudo yum install -y amazon-efs-utils` (AL2) or `sudo apt install -y amazon-efs-utils` (Ubuntu)

SHOULD:
- Use the mount helper: `sudo mount -t efs <fs-id>:/ /mnt/efs`
- Add to /etc/fstab for persistent mount

MAY:
- Enable mount helper logging for debugging

## Common Issues

- symptoms: "mount.nfs: Connection timed out"
  diagnosis: "Security group is blocking NFS traffic on port 2049."
  resolution: "Add inbound TCP 2049 rule to mount target SG from the instance SG."

- symptoms: "mount.nfs: No such file or directory"
  diagnosis: "The mount point directory doesn't exist."
  resolution: "Create the mount point: mkdir -p /mnt/efs"

- symptoms: "mount: /efs: bad option; for several filesystems (e.g. nfs, cifs)"
  diagnosis: "NFS client or amazon-efs-utils not installed."
  resolution: "Install amazon-efs-utils or nfs-utils package."

## Output Format

```yaml
root_cause: "mount_failure â€” <specific_cause>"
evidence:
  - type: mount_targets
    content: "<mount target configuration>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix mount target, security group, or install NFS client"
  long_term: "Automate mount target creation and security group configuration"
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
