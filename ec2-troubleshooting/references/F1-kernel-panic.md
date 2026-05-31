---
title: "F1 — Kernel Panic"
description: "Diagnose and recover from kernel panic on EC2 instances"
status: active
severity: CRITICAL
triggers:
  - "Kernel panic"
  - "not syncing"
  - "VFS.*Unable to mount root"
  - "end Kernel panic"
owner: devops-agent
objective: "Identify the kernel panic cause and restore the instance to a bootable state"
context: "Kernel panics prevent the instance from booting. Common causes: corrupt kernel, missing initramfs, incompatible kernel modules, root filesystem not found, driver issues after instance type change. Visible in system log (get-console-output)."
---

## Phase 1 — Triage

MUST:
- Get system log: `aws ec2 get-console-output --instance-id <id>` — find the panic message
- Get screenshot: `aws ec2 get-console-screenshot --instance-id <id>` — visual state
- Classify the panic:
  - "VFS: Unable to mount root fs" → root device not found (wrong device name or missing driver)
  - "Kernel panic - not syncing: No init found" → corrupt root filesystem
  - "BUG: unable to handle kernel paging request" → kernel bug or memory corruption

SHOULD:
- Check if instance type was recently changed (Xen → Nitro requires NVMe drivers)
- Check if kernel was recently updated

## Phase 2 — Remediate

MUST:
- Stop the instance
- Detach root volume
- Attach to a rescue instance (same OS family)
- Mount the volume and investigate/fix:
  - Missing NVMe drivers: install nvme module in initramfs
  - Corrupt kernel: chroot and reinstall kernel package
  - Wrong root device: update GRUB config
- Detach from rescue, reattach as root to original instance, start

## Common Issues

- symptoms: "VFS: Unable to mount root fs on unknown-block(0,0)"
  diagnosis: "Root device not found. Common when migrating from Xen (xvda) to Nitro (nvme) instance types."
  resolution: "Rescue instance: install NVMe drivers, rebuild initramfs, update /etc/fstab to use UUID instead of device name."

- symptoms: "Kernel panic after yum/apt kernel update"
  diagnosis: "New kernel incompatible or corrupt initramfs."
  resolution: "Rescue instance: set previous kernel as default in GRUB, or rebuild initramfs for new kernel."

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
  - "Snapshot root volume before repair: GREEN — non-destructive backup"
  - "Detach/attach root volume to rescue instance: YELLOW — careful device mapping required"
  - "Install NVMe drivers in initramfs via chroot: YELLOW — modifies boot configuration"
  - "Reinstall kernel package via chroot: YELLOW — replaces kernel files, keep old kernel as fallback"
  - "Update GRUB config via chroot: YELLOW — modifies boot loader, recoverable from rescue"
```

## Escalation Conditions
- Kernel panic persists after rescue instance repair attempt
- Root volume cannot be mounted on rescue instance (filesystem corruption)
- Instance is in a production Auto Scaling group with no healthy instances
- Kernel panic was caused by a kernel update applied across a fleet of instances
- Panic message indicates hardware/memory corruption (not software)
- Instance uses encrypted root volume and rescue instance lacks KMS permissions

## Data Sensitivity
- HIGH: get-console-output (contains kernel messages, may include filesystem paths and driver details)
- HIGH: Root volume contents when mounted on rescue instance (full filesystem access)
- MEDIUM: get-console-screenshot (visual state of boot process)

## Prohibited Actions
- NEVER suggest terminating an instance to fix a kernel panic — use rescue instance approach
- NEVER suggest modifying initramfs without first snapshotting the root volume
- NEVER suggest deleting old kernel packages during repair (keep as fallback)
- NEVER suggest changing instance type during kernel panic recovery (fix boot first)

## Phase 3 — Rollback
- If kernel was reinstalled: boot with previous kernel by selecting it in GRUB menu or setting default
- If initramfs was rebuilt: restore root volume from pre-repair snapshot
- If GRUB config was modified: restore from snapshot or manually revert GRUB configuration
- If NVMe drivers were added: revert to original initramfs from snapshot if drivers cause issues
- Always keep the pre-repair snapshot until the instance is confirmed stable

## Output Format

```yaml
root_cause: "<missing_driver|corrupt_kernel|wrong_root_device|initramfs> — <detail>"
evidence:
  - type: system_log
    content: "<kernel panic message from console output>"
severity: CRITICAL
mitigation:
  immediate: "Rescue instance repair"
  long_term: "Test kernel updates in staging, use UUID in fstab, keep previous kernel"
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
