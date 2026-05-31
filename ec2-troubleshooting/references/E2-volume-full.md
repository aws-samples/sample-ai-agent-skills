---
title: "E2 — Volume Full / Disk Space Exhaustion"
description: "Diagnose and remediate disk space exhaustion on EBS volumes"
status: active
severity: HIGH
triggers:
  - "No space left on device"
  - "disk.*full"
  - "DiskSpaceUtilization.*100"
  - "inode.*exhaustion"
owner: devops-agent
objective: "Free disk space or expand the volume to restore operations"
context: "Disk full can be caused by log accumulation, temp files, large deployments, or inode exhaustion (many small files). EBS volumes can be expanded online without downtime (Nitro instances). After expansion, the filesystem must be grown to use the new space."
---

## Phase 1 — Triage

MUST:
- If SSM available: `df -h` (space), `df -i` (inodes), `du -sh /* | sort -rh | head -20` (largest directories)
- Check volume size: `aws ec2 describe-volumes --volume-ids <vol-id>`
- Check system log for "No space left on device" errors

SHOULD:
- Identify the largest space consumers: logs, temp files, old deployments, core dumps
- Check if inode exhaustion (many small files) vs block exhaustion (large files)

MAY:
- Check CloudWatch Agent disk metrics if installed

## Phase 2 — Remediate

MUST:
- Free space immediately: clean logs, temp files, old packages
  - `journalctl --vacuum-size=100M`
  - `find /tmp -type f -mtime +7 -delete`
  - `yum clean all` or `apt-get clean`
- If more space needed: expand EBS volume online
  - `aws ec2 modify-volume --volume-id <vol-id> --size <new-size>`
  - Wait for modification to complete: `aws ec2 describe-volumes-modifications --volume-ids <vol-id>`
  - Grow filesystem: `growpart /dev/xvda 1 && resize2fs /dev/xvda1` (ext4) or `xfs_growfs /` (xfs)

SHOULD:
- Set up log rotation to prevent recurrence
- Configure CloudWatch alarm on disk usage > 80%

## Common Issues

- symptoms: "df -h shows 100% but du shows less total"
  diagnosis: "Deleted files still held open by processes. Space not freed until process releases the file handle."
  resolution: "`lsof +L1` to find deleted-but-open files. Restart the process holding them."

- symptoms: "df -i shows 100% inodes used"
  diagnosis: "Inode exhaustion from many small files (common with mail queues, session files, cache)."
  resolution: "Find and clean small file accumulations. Inode count is set at filesystem creation."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "df, du, lsof via SSM: GREEN — read-only diagnostics"
  - "describe-volumes: GREEN — read-only"
  - "Clean temp files and old logs: YELLOW — recoverable if files are backed up"
  - "journalctl --vacuum-size: YELLOW — removes old journal entries, non-recoverable but low risk"
  - "Expand EBS volume (modify-volume): YELLOW — online operation, cannot shrink back"
  - "Grow filesystem (resize2fs/xfs_growfs): YELLOW — extends filesystem, generally safe"
  - "Delete files with find -delete: RED — permanent file deletion, not recoverable without backup"
```

## Escalation Conditions
- Root volume is full and instance is unresponsive (cannot SSH or SSM)
- Disk full is causing database corruption or transaction log failures
- Volume is at maximum EBS size (64 TiB) and cannot be expanded further
- Inode exhaustion requires filesystem recreation (cannot add inodes to existing ext4)
- Disk full is caused by a security incident (unauthorized data exfiltration staging)

## Data Sensitivity
- HIGH: du/find output via SSM (reveals file paths, directory structure, file sizes)
- HIGH: lsof output (reveals open files, process names, file descriptors)
- MEDIUM: describe-volumes (reveals volume size, type, encryption status)
- MEDIUM: CloudWatch disk metrics (reveals usage patterns)

## Prohibited Actions
- NEVER suggest deleting files in /var/lib or /var/log without understanding what they contain
- NEVER suggest reducing EBS volume size (EBS volumes cannot be shrunk)
- NEVER suggest running resize2fs on a mounted root filesystem without confirming online resize support
- NEVER suggest deleting core dumps without first analyzing them for crash root cause

## Phase 3 — Rollback
- If files were deleted: restore from backup or snapshot if deletion was premature
- If volume was expanded: cannot shrink back — create smaller volume from snapshot if needed
- If filesystem was grown: cannot shrink — this is a one-way operation
- If log rotation was configured: revert logrotate config if rotation is too aggressive
- If journal was vacuumed: journal entries are permanently removed, no rollback possible

## Output Format

```yaml
root_cause: "<log_accumulation|large_files|inode_exhaustion|deleted_open_files> — <detail>"
evidence:
  - type: disk_usage
    content: "<df -h and du output>"
severity: HIGH
mitigation:
  immediate: "Clean space or expand volume"
  long_term: "Log rotation, disk monitoring alerts, auto-scaling storage"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
