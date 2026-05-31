---
title: "D2 — Memory Pressure / OOM"
description: "Diagnose memory exhaustion and OOM kills on EC2 instances"
status: active
severity: HIGH
triggers:
  - "Out of memory"
  - "oom-killer"
  - "Cannot allocate memory"
  - "swap.*full"
owner: devops-agent
objective: "Identify the memory-consuming process and restore instance stability"
context: "EC2 does not have a native CloudWatch memory metric. Memory monitoring requires CloudWatch Agent or SSM. OOM kills are visible in system log (dmesg). Linux OOM killer selects processes based on oom_score."
---

## Phase 1 — Triage

MUST:
- CloudWatch Agent memory metric (if installed): `aws cloudwatch get-metric-statistics --namespace CWAgent --metric-name mem_used_percent`
- System log for OOM: `aws ec2 get-console-output --instance-id <id>` — search for "oom-killer" or "Out of memory"
- If SSM available: `free -m`, `cat /proc/meminfo`, `ps aux --sort=-%mem | head -20`

SHOULD:
- Check swap usage: `swapon --show` via SSM
- Check dmesg for OOM events: `dmesg | grep -i oom` via SSM
- Review memory trend if CloudWatch Agent is installed

MAY:
- Check /proc/<pid>/oom_score for critical processes
- Review /var/log/messages or journalctl for OOM history

## Phase 2 — Remediate

MUST:
- Identify the memory-consuming process and determine if it's a leak or legitimate usage
- If memory leak: restart the process as immediate fix, investigate root cause
- If legitimate usage: upgrade to instance type with more memory

SHOULD:
- Add swap space as a buffer (not a permanent fix)
- Configure OOM killer priorities to protect critical processes
- Install CloudWatch Agent for memory monitoring

## Common Issues

- symptoms: "Instance unresponsive, system log shows oom-killer"
  diagnosis: "Memory exhausted, kernel killed processes. Check which process was killed and why."
  resolution: "Upgrade instance type, fix memory leak, add swap as temporary buffer."

- symptoms: "No CloudWatch memory metric available"
  diagnosis: "CloudWatch Agent not installed. EC2 does not report memory natively."
  resolution: "Install and configure CloudWatch Agent for memory, disk, and process metrics."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "CloudWatch get-metric-statistics for memory: GREEN — read-only"
  - "get-console-output to check for OOM: GREEN — read-only"
  - "SSM commands (free, ps, dmesg): GREEN — read-only diagnostics"
  - "Restart memory-leaking process: YELLOW — brief service interruption, recoverable"
  - "Add swap space via SSM: YELLOW — temporary fix, recoverable by removing swap"
  - "Upgrade instance type for more memory: YELLOW — requires stop+start, recoverable"
  - "Configure OOM killer priorities: YELLOW — changes kernel behavior, recoverable by resetting"
```

## Escalation Conditions
- OOM killer is repeatedly killing critical production processes
- Memory exhaustion is caused by a suspected security incident (malware, unauthorized process)
- Instance is completely unresponsive and SSM cannot connect
- Multiple instances simultaneously experiencing OOM (possible application-wide memory leak)
- Memory leak is in a shared library or kernel module affecting system stability

## Data Sensitivity
- HIGH: get-console-output (may contain OOM killer output with process names, PIDs, memory maps)
- HIGH: SSM command output from ps/top (reveals running processes, memory usage, command arguments)
- HIGH: /proc/meminfo, dmesg output (may contain kernel-level details and process information)
- MEDIUM: CloudWatch Agent memory metrics (reveals capacity utilization patterns)

## Prohibited Actions
- NEVER suggest terminating an instance to fix memory pressure — restart the process or upgrade instead
- NEVER suggest disabling the OOM killer entirely (system will hang instead of recovering)
- NEVER suggest using swap on EBS as a permanent solution for memory-intensive workloads
- NEVER suggest reducing memory limits for critical system processes

## Phase 3 — Rollback
- If instance type was upgraded: stop instance, change back to original type, restart
- If swap was added: remove swap file/partition with `swapoff` and delete the file
- If OOM killer priorities were changed: reset oom_score_adj to 0 for affected processes
- If process was restarted: monitor for memory leak recurrence, restore previous process configuration
- If CloudWatch Agent was installed: uninstall if not desired with package manager

## Output Format

```yaml
root_cause: "<memory_leak|undersized|no_swap|oom_kill> — <process_name>"
evidence:
  - type: system_log
    content: "<OOM killer output from dmesg>"
severity: HIGH
mitigation:
  immediate: "Restart process, add swap, or upgrade instance"
  long_term: "Fix memory leak, install CloudWatch Agent, configure OOM priorities"
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
