---
title: "B2 — Instance Status Check Failure"
description: "Diagnose and remediate EC2 instance status check failures caused by OS or software issues"
status: active
severity: CRITICAL
triggers:
  - "StatusCheckFailed_Instance"
  - "instance status check.*fail"
  - "impaired.*instance"
owner: devops-agent
objective: "Identify the OS-level issue causing the instance status check failure and restore the instance"
context: "Instance status checks monitor the OS-level health by verifying the instance responds to ARP requests on its network interface. Failures indicate kernel panic, misconfigured networking inside the OS, corrupted filesystem, exhausted memory, or failed boot."
---

## Phase 1 — Triage

MUST:
- Confirm instance status check failure: `aws ec2 describe-instance-status --instance-ids <id>`
- Get system log: `aws ec2 get-console-output --instance-id <id>` — look for kernel panic, fsck errors, OOM, network config errors
- Get screenshot: `aws ec2 get-console-screenshot --instance-id <id>` — visual boot state (GRUB, fsck prompt, login screen, BSOD)
- Check if system status check is passing (confirms host is healthy, problem is inside the instance)

SHOULD:
- Search system log for: kernel panic, OOM, filesystem errors, network interface failures, cloud-init errors
- Check recent changes: was the instance rebooted, was user-data changed, was a new AMI deployed?
- Check CloudWatch metrics before failure: CPU spike, memory pressure, disk full

MAY:
- If instance has SSM Agent and it's responsive: run diagnostics via SSM
- Check CloudTrail for recent API calls that modified the instance (modify-instance-attribute, etc.)

## Phase 2 — Remediate

MUST:
- Based on system log analysis, identify the root cause category:
  - Kernel panic → See F1 (kernel panic runbook)
  - Filesystem corruption → See F2 (fsck runbook)
  - Network misconfiguration → Fix network config via rescue instance
  - Memory exhaustion → Increase instance size or fix memory leak
  - Failed boot → See F3/F4 (boot failure runbooks)
- If OS is unrecoverable: detach root volume, attach to rescue instance, fix, reattach

SHOULD:
- Try a reboot first (if system log suggests a transient issue)
- If reboot doesn't help: stop, detach root volume, attach to rescue instance for repair

MAY:
- Launch a new instance from the same AMI and migrate data from the old root volume

## Guardrails

escalation_conditions:
  - "System log shows no output (instance may not be booting at all)"
  - "Root volume cannot be detached (instance won't stop)"
  - "Rescue instance approach fails to identify the issue"

safety_ratings:
  - "get-console-output, get-console-screenshot, describe-instance-status: GREEN (read-only)"
  - "Reboot instance: YELLOW — may cause brief downtime"
  - "Detach root volume for rescue: YELLOW — requires instance stop"

## Common Issues

- symptoms: "System log shows kernel panic with stack trace"
  diagnosis: "Kernel crash, possibly due to driver incompatibility, corrupt kernel, or hardware-triggered bug."
  resolution: "See F1 runbook. May need to boot with older kernel via rescue instance."

- symptoms: "Screenshot shows fsck prompt waiting for input"
  diagnosis: "Filesystem corruption detected during boot. Instance is waiting for manual fsck."
  resolution: "See F2 runbook. Detach root volume, attach to rescue instance, run fsck, reattach."

- symptoms: "System log shows network interface not found or DHCP failure"
  diagnosis: "OS network configuration doesn't match the instance's ENI. Common after AMI migration or manual network config changes."
  resolution: "Detach root volume, mount on rescue instance, fix /etc/sysconfig/network-scripts/ or /etc/netplan/ config."

- symptoms: "System log shows OOM killer invoked repeatedly"
  diagnosis: "Instance ran out of memory. Processes killed by OOM killer, including critical services."
  resolution: "Increase instance size (more memory) or fix the memory-consuming process. Add swap as temporary measure."

## Output Format

```yaml
root_cause: "<kernel_panic|filesystem_corruption|network_misconfig|oom|boot_failure> — <detail>"
evidence:
  - type: system_log
    content: "<relevant excerpt from get-console-output>"
  - type: screenshot
    content: "<description of get-console-screenshot>"
severity: CRITICAL
mitigation:
  immediate: "Reboot or rescue instance repair"
  long_term: "Fix root cause, update AMI, configure monitoring"
```

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
