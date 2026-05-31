---
title: "F2 — Filesystem Corruption / fsck"
description: "Diagnose and repair filesystem corruption preventing boot"
status: active
severity: HIGH
triggers:
  - "fsck.*error"
  - "UNEXPECTED INCONSISTENCY"
  - "filesystem.*corrupt"
  - "superblock.*invalid"
owner: devops-agent
objective: "Repair the corrupted filesystem and restore the instance to bootable state"
context: "Filesystem corruption can occur from unclean shutdown, EBS volume issues, or disk errors. The instance may hang at boot waiting for manual fsck input. Screenshot will show the fsck prompt."
---

## Phase 1 — Triage

MUST:
- Get screenshot: `aws ec2 get-console-screenshot --instance-id <id>` — look for fsck prompt
- Get system log: `aws ec2 get-console-output --instance-id <id>` — look for filesystem errors
- Identify which filesystem/partition is corrupted

## Phase 2 — Remediate

MUST:
- Stop the instance
- Snapshot the root volume (backup before repair)
- Detach root volume, attach to rescue instance
- Run fsck: `fsck -y /dev/xvdf1` (or appropriate device)
- Reattach as root volume, start instance

SHOULD:
- Check /etc/fstab for incorrect entries that could cause boot failure
- Remove fsck.mode=force from kernel parameters if set

## Common Issues

- symptoms: "Screenshot shows 'Press F to fix' or fsck prompt"
  diagnosis: "Filesystem corruption detected during boot. Instance waiting for manual input."
  resolution: "Rescue instance: run `fsck -y` on the affected partition."

- symptoms: "System log shows 'UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY'"
  diagnosis: "Severe filesystem corruption requiring manual repair."
  resolution: "Rescue instance: snapshot first, then `fsck -y`. If fsck fails, restore from snapshot."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "get-console-output, get-console-screenshot: GREEN — read-only diagnostics"
  - "Stop instance for rescue: YELLOW — causes downtime, changes public IP"
  - "Snapshot root volume before fsck: GREEN — critical backup before repair"
  - "Detach/attach root volume to rescue instance: YELLOW — careful device mapping required"
  - "Run fsck -y on filesystem: YELLOW — may modify filesystem structure, snapshot first"
  - "Fix /etc/fstab entries: YELLOW — incorrect changes can prevent boot"
```

## Escalation Conditions
- fsck reports unrecoverable errors or fails to repair the filesystem
- Multiple partitions are corrupted simultaneously
- Filesystem corruption is recurring (may indicate underlying EBS volume issue)
- Root volume snapshot fails (volume may be in inconsistent state)
- Instance is in a production environment with no recent backups
- Corruption affects a database data directory

## Data Sensitivity
- HIGH: get-console-output (contains filesystem error details, partition information)
- HIGH: Root volume contents when mounted on rescue instance (full filesystem access)
- HIGH: fsck output (reveals file paths, inode details, directory structure)
- MEDIUM: get-console-screenshot (visual state showing fsck prompt)

## Prohibited Actions
- NEVER run fsck on a mounted filesystem (must unmount first)
- NEVER skip snapshotting the volume before running fsck
- NEVER suggest deleting the corrupted volume before confirming snapshot is complete
- NEVER suggest reformatting the filesystem as a first resort (data loss)

## Phase 3 — Rollback
- If fsck made things worse: restore root volume from pre-repair snapshot
- If /etc/fstab was modified incorrectly: re-attach to rescue instance and fix fstab
- If filesystem cannot be repaired: create new volume from last known-good snapshot
- Always keep the pre-repair snapshot until the instance is confirmed stable and data verified

## Output Format

```yaml
root_cause: "<unclean_shutdown|disk_error|fstab_error> — filesystem corruption on <partition>"
evidence:
  - type: system_log
    content: "<fsck error messages>"
severity: HIGH
mitigation:
  immediate: "Rescue instance fsck repair"
  long_term: "Use journaling filesystem, configure auto-fsck, regular snapshots"
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
