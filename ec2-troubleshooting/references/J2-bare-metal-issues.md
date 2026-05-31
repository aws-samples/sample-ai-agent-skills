---
title: "J2 — Bare Metal Instance Issues"
description: "Diagnose issues specific to bare metal EC2 instances"
status: active
severity: MEDIUM
triggers:
  - "bare metal.*boot"
  - "metal.*slow start"
  - "BIOS.*configuration"
owner: devops-agent
objective: "Resolve bare metal specific issues"
context: "Bare metal instances (.metal suffix) provide direct hardware access. They take longer to launch (5-20 minutes), have BIOS/UEFI access, and expose all hardware features. No hypervisor overhead but also no hypervisor-level protections."
---

## Phase 1 — Triage

MUST:
- Confirm instance type is bare metal: `aws ec2 describe-instances --instance-ids <id>` → InstanceType (*.metal)
- Check if slow launch is expected (bare metal takes 5-20 minutes to boot)
- Check system log for BIOS/UEFI messages

SHOULD:
- Verify AMI supports UEFI boot mode if required by the bare metal type
- Check if NVMe instance store volumes need initialization

## Common Issues

- symptoms: "Bare metal instance takes 15+ minutes to reach running state"
  diagnosis: "Normal behavior. Bare metal instances perform full hardware initialization."
  resolution: "This is expected. Plan for longer launch times in automation."

- symptoms: "Instance store NVMe volumes not visible"
  diagnosis: "Instance store volumes may need to be initialized/formatted on first use."
  resolution: "Use `lsblk` to find NVMe devices, then format and mount."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances, describe-instance-types: GREEN — read-only"
  - "Check system log for BIOS/UEFI messages: GREEN — read-only"
  - "lsblk to find NVMe devices via SSM: GREEN — read-only"
  - "Format and mount instance store NVMe volumes: YELLOW — initializes storage, data loss if wrong device"
  - "Modify BIOS/UEFI settings: YELLOW — changes hardware configuration, recoverable"
```

## Escalation Conditions
- Bare metal instance fails to boot after 20+ minutes
- BIOS/UEFI configuration issue requires serial console access
- Instance store NVMe volumes are not detected by the OS
- Bare metal instance has hardware-level issues not visible in system log
- Performance is significantly different from expected bare metal specifications

## Data Sensitivity
- MEDIUM: system log (contains BIOS/UEFI initialization messages, hardware details)
- MEDIUM: describe-instances (reveals instance type and hardware configuration)
- LOW: lsblk output (reveals block device layout)

## Prohibited Actions
- NEVER suggest treating slow bare metal boot as an error (5-20 minutes is normal)
- NEVER suggest formatting NVMe devices without confirming they are instance store (not EBS)
- NEVER suggest modifying BIOS settings without understanding the impact on boot
- NEVER suggest using bare metal instances for workloads that don't require direct hardware access

## Phase 3 — Rollback
- If NVMe volumes were formatted: data on those volumes is lost — restore from backup if needed
- If BIOS/UEFI settings were changed: revert settings via serial console or rescue approach
- If instance type was changed from bare metal: stop, change back to .metal type, start
- If AMI was modified for UEFI support: revert to previous AMI version

## Output Format

```yaml
root_cause: "<slow_boot_expected|bios_config|nvme_init|uefi_required>"
evidence:
  - type: system_log
    content: "<boot messages>"
severity: MEDIUM
mitigation:
  immediate: "Wait for boot or fix configuration"
  long_term: "Pre-warm AMIs for bare metal, automate NVMe initialization"
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
