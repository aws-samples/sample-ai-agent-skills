---
title: "F4 — GRUB / Boot Loader Failures"
description: "Diagnose boot loader failures preventing instance startup"
status: active
severity: CRITICAL
triggers:
  - "GRUB.*error"
  - "error.*loading.*kernel"
  - "Booting from Hard Disk"
  - "grub rescue"
owner: devops-agent
objective: "Repair the boot loader and restore the instance to a bootable state"
context: "GRUB failures prevent the kernel from loading. Common causes: corrupt GRUB config, missing kernel files, wrong root device in GRUB, or GRUB installation corruption. Screenshot shows GRUB menu or rescue prompt."
---

## Phase 1 — Triage

MUST:
- Get screenshot: `aws ec2 get-console-screenshot --instance-id <id>` — look for GRUB prompt or error
- Get system log: `aws ec2 get-console-output --instance-id <id>` — look for GRUB messages
- Identify the GRUB error type: missing kernel, wrong root, corrupt config

## Phase 2 — Remediate

MUST:
- Stop instance, snapshot root volume
- Attach root volume to rescue instance
- Mount and chroot into the filesystem
- Fix GRUB config: `grub2-mkconfig -o /boot/grub2/grub.cfg` (RHEL/AL2) or `update-grub` (Ubuntu)
- Reinstall GRUB if needed: `grub2-install /dev/xvdf` (adjust device)
- Detach, reattach as root, start

## Common Issues

- symptoms: "Screenshot shows 'grub rescue>' prompt"
  diagnosis: "GRUB cannot find its configuration or modules. Partition table or GRUB install corrupted."
  resolution: "Rescue instance: reinstall GRUB and regenerate config."

- symptoms: "GRUB shows 'error: file not found' for kernel"
  diagnosis: "Kernel file missing from /boot. May have been deleted or filesystem corrupted."
  resolution: "Rescue instance: chroot and reinstall kernel package."

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
  - "Snapshot root volume before repair: GREEN — critical backup"
  - "Regenerate GRUB config via chroot: YELLOW — modifies boot loader configuration"
  - "Reinstall GRUB (grub2-install): YELLOW — rewrites boot sector, snapshot first"
  - "Reinstall kernel package via chroot: YELLOW — replaces kernel files"
```

## Escalation Conditions
- GRUB repair fails and instance still cannot boot
- Root volume cannot be mounted on rescue instance
- GRUB failure was caused by a fleet-wide kernel update
- Instance uses UEFI boot mode and standard GRUB repair steps don't apply
- Boot failure is on an instance with encrypted root volume requiring KMS access
- Multiple boot issues compound (GRUB + filesystem corruption)

## Data Sensitivity
- HIGH: get-console-output (contains boot loader messages, kernel parameters, root device paths)
- HIGH: Root volume contents when mounted on rescue instance (full filesystem access)
- HIGH: GRUB configuration (may contain kernel parameters with security-relevant settings)
- MEDIUM: get-console-screenshot (visual state of GRUB error)

## Prohibited Actions
- NEVER suggest reinstalling GRUB without first snapshotting the root volume
- NEVER suggest deleting kernel files from /boot during repair
- NEVER suggest modifying partition tables to fix GRUB issues
- NEVER suggest converting between BIOS and UEFI boot modes as a fix

## Phase 3 — Rollback
- If GRUB config was regenerated: restore from pre-repair snapshot if new config is incorrect
- If GRUB was reinstalled: restore from snapshot if boot sector is corrupted
- If kernel was reinstalled: select previous kernel in GRUB menu or restore from snapshot
- Always keep the pre-repair snapshot until the instance is confirmed bootable and stable

## Output Format

```yaml
root_cause: "<corrupt_grub|missing_kernel|wrong_root_device|grub_install> — <detail>"
evidence:
  - type: screenshot
    content: "<GRUB error from screenshot>"
severity: CRITICAL
mitigation:
  immediate: "Rescue instance GRUB repair"
  long_term: "Keep backup kernel, test updates in staging"
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
