---
title: "J1 — Nitro vs Xen Instance Migration Issues"
description: "Diagnose issues when migrating between Xen and Nitro instance types"
status: active
severity: HIGH
triggers:
  - "NVMe.*not found"
  - "xvd.*missing"
  - "ENA.*not supported"
  - "instance type change.*boot failure"
owner: devops-agent
objective: "Successfully migrate between Xen and Nitro instance types"
context: "Nitro instances use NVMe for EBS and ENA for networking. Xen instances use Xen virtual block devices (xvd*) and legacy networking. Migrating requires NVMe and ENA drivers in the AMI. Device names change: /dev/xvda → /dev/nvme0n1."
---

## Phase 1 — Triage

MUST:
- Check if the AMI has NVMe drivers: look for nvme module in initramfs
- Check if ENA is enabled on the AMI: `aws ec2 describe-images --image-ids <ami-id>` → EnaSupport
- Check /etc/fstab for device name references (must use UUID, not /dev/xvd*)
- Check system log for NVMe or ENA errors after instance type change

## Phase 2 — Remediate

MUST:
- Before migration: install NVMe drivers and ENA driver on the source instance
- Update /etc/fstab to use UUID or LABEL instead of device names
- Enable ENA on the AMI: `aws ec2 modify-instance-attribute --instance-id <id> --ena-support`
- Test by launching a new Nitro instance from the AMI before migrating production

## Common Issues

- symptoms: "Kernel panic after changing to Nitro instance type"
  diagnosis: "AMI lacks NVMe drivers. Root device not found as /dev/nvme0n1."
  resolution: "Change back to Xen type, install NVMe drivers, rebuild initramfs, then migrate."

- symptoms: "Instance launches but no network on Nitro"
  diagnosis: "ENA not enabled on the AMI or ENA driver not installed."
  resolution: "Install ENA driver, enable ENA support on the instance/AMI."

- symptoms: "Boot fails with 'unable to mount root' after migration"
  diagnosis: "/etc/fstab references /dev/xvda1 which doesn't exist on Nitro (it's /dev/nvme0n1p1)."
  resolution: "Use rescue instance to update /etc/fstab to use UUID."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-images EnaSupport, describe-instance-types: GREEN — read-only"
  - "Check system log for NVMe/ENA errors: GREEN — read-only"
  - "Install NVMe/ENA drivers on source instance: YELLOW — modifies OS, recoverable"
  - "Rebuild initramfs: YELLOW — modifies boot configuration, keep old initramfs as backup"
  - "Update /etc/fstab to use UUID: YELLOW — modifies boot configuration, recoverable"
  - "Enable ENA support on instance: YELLOW — changes instance attribute, recoverable"
  - "Change instance type back to Xen: YELLOW — requires stop+start, recoverable"
```

## Escalation Conditions
- Migration failure affects a production instance that cannot be reverted to Xen type
- AMI lacks NVMe drivers and rescue instance approach is needed
- Fleet-wide migration is planned and test instance failed
- Instance has custom kernel or drivers that may not support Nitro
- Migration involves encrypted volumes with cross-account KMS keys

## Data Sensitivity
- HIGH: system log after migration (contains boot errors, driver details, device mapping)
- HIGH: /etc/fstab contents (reveals filesystem layout and mount points)
- MEDIUM: describe-images (reveals AMI configuration and driver support)

## Prohibited Actions
- NEVER suggest migrating to Nitro without first verifying NVMe and ENA driver support
- NEVER suggest changing /etc/fstab device names without using UUID or LABEL
- NEVER suggest migrating production instances without testing on a non-production copy first
- NEVER suggest removing Xen drivers after migration (keep for rollback capability)

## Phase 3 — Rollback
- If instance fails to boot on Nitro: stop instance, change back to Xen instance type, start
- If /etc/fstab was modified incorrectly: use rescue instance to fix fstab
- If initramfs was rebuilt incorrectly: boot with old initramfs or use rescue instance to restore
- If ENA was enabled but causes issues: disable with `modify-instance-attribute --no-ena-support`

## Output Format

```yaml
root_cause: "<missing_nvme_driver|missing_ena|fstab_device_names|ena_not_enabled>"
evidence:
  - type: system_log
    content: "<boot error from console output>"
severity: HIGH
mitigation:
  immediate: "Revert to previous instance type, fix drivers/config, then migrate"
  long_term: "Always use UUID in fstab, include NVMe+ENA in base AMIs"
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
