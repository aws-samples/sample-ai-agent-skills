---
title: "B1 — Driver Installation"
description: "Diagnose EFA driver installation and kernel module issues"
status: active
severity: HIGH
triggers: ["efa driver not found", "driver installation failed", "kernel module missing", "efa_verbs not loaded"]
owner: devops-agent
objective: "Resolve EFA driver installation failures and ensure kernel modules are loaded"
context: "EFA requires the efa kernel module, rdma-core, and libfabric. These are installed via the AWS EFA installer package."
---

## Phase 1 — Triage

MUST:
- Check if EFA kernel module is loaded: `lsmod | grep efa`
- Verify EFA device is present: `ls -la /dev/infiniband/`
- Check installed EFA components: `fi_info --version`

SHOULD:
- Check kernel version compatibility: `uname -r`
- Verify the EFA installer was run: `rpm -qa | grep efa` or `dpkg -l | grep efa`

MAY:
- Check dmesg for EFA driver errors: `dmesg | grep -i efa`
- Review the EFA installer log: `cat /tmp/efa_installer.log`

## Phase 2 — Remediate

MUST:
- Download and install the EFA installer: `curl -O https://efa-installer.amazonaws.com/aws-efa-installer-latest.tar.gz && tar xf aws-efa-installer-latest.tar.gz && cd aws-efa-installer && sudo ./efa_installer.sh -y`
- Load the EFA kernel module: `sudo modprobe efa`
- Verify installation: `fi_info -p efa`

SHOULD:
- Install with GPU support if needed: `sudo ./efa_installer.sh -y -g` (includes GDR support)
- Add EFA module to load on boot: ensure efa is in /etc/modules-load.d/

MAY:
- Build a custom AMI with EFA pre-installed for faster instance launches

## Safety Ratings

```
safety_ratings:
  - "lsmod | grep efa: GREEN — read-only kernel module check"
  - "fi_info -p efa: GREEN — read-only provider enumeration"
  - "efa_installer.sh -y: YELLOW — installs kernel modules and libraries"
  - "modprobe efa: YELLOW — loads kernel module, may affect running workloads"
```

## Escalation Conditions

- EFA installer fails with kernel header mismatch
- Kernel module loads but EFA device does not appear in /dev/infiniband/
- Driver installation conflicts with existing NVIDIA or CUDA drivers
- Custom kernel prevents EFA module compilation

## Data Sensitivity

- `lsmod` output: LOW — lists loaded kernel modules
- `fi_info` output: LOW — lists available fabric providers
- `dmesg` output: MEDIUM — may contain system configuration details
- EFA installer log: LOW — installation steps and versions

## Prohibited Actions

- NEVER install EFA drivers on instances without EFA interfaces attached
- NEVER downgrade the kernel to match an older EFA driver version
- NEVER unload the EFA module while workloads are actively using it
- NEVER install untrusted or unofficial EFA driver packages

## Phase 3 — Rollback

1. Uninstall EFA components: `cd aws-efa-installer && sudo ./efa_installer.sh -u`
2. Remove the EFA kernel module: `sudo modprobe -r efa`
3. If kernel was updated, reboot to previous kernel version
4. Verify standard ENA networking still functions after EFA removal
5. Document the driver version and kernel version that caused the issue

## Common Issues

- symptoms: "efa_installer.sh fails with 'kernel headers not found'"
  diagnosis: "Kernel development headers are not installed for the running kernel."
  resolution: "Install kernel headers: sudo yum install kernel-devel-$(uname -r) or sudo apt install linux-headers-$(uname -r)."

- symptoms: "fi_info -p efa returns no results but driver is installed"
  diagnosis: "EFA interface is not attached or the device is not recognized by the kernel."
  resolution: "Verify EFA interface is attached. Check dmesg for device recognition. Reload module: sudo modprobe -r efa && sudo modprobe efa."

## Output Format

```yaml
root_cause: "driver — <specific_cause>"
evidence:
  - type: driver_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Install or reload EFA driver"
  long_term: "Bake EFA driver into custom AMI"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
