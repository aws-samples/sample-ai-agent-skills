---
title: "D1 — CPU Exhaustion"
description: "Diagnose high CPU utilization causing instance unresponsiveness or degraded performance"
status: active
severity: HIGH
triggers:
  - "CPUUtilization.*100"
  - "load average.*high"
  - "instance.*slow"
  - "unresponsive"
owner: devops-agent
objective: "Identify the CPU-consuming process and restore normal performance"
context: "Sustained 100% CPU causes instance unresponsiveness, SSH timeouts, and application failures. For T-series instances, CPU credits may be exhausted. For fixed-performance instances, the workload exceeds the instance's compute capacity."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch CPUUtilization: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=<id> --period 300 --statistics Average --start-time <time> --end-time <time>`
- For T-series: check CPUCreditBalance and CPUSurplusCreditBalance metrics
- If SSM available: `top -bn1 | head -20` to identify the process consuming CPU
- Check instance type vCPU count vs workload requirements

SHOULD:
- Check if CPU spike correlates with a specific event (deployment, cron job, traffic spike)
- Review CloudWatch CPUUtilization trend over the past 24 hours

MAY:
- Enable detailed monitoring (1-minute intervals) for better granularity
- Check steal time (st in top) — high steal indicates noisy neighbor on shared tenancy

## Phase 2 — Remediate

MUST:
- Identify and address the CPU-consuming process
- For T-series credit exhaustion: switch to unlimited mode or upgrade instance type
- For sustained high CPU: vertical scale (larger instance type) or horizontal scale (add instances)

SHOULD:
- Kill runaway processes if they're not critical
- Optimize application code or queries causing high CPU

## Common Issues

- symptoms: "CPUUtilization at 100%, T2 instance, CPUCreditBalance at 0"
  diagnosis: "T2 CPU credits exhausted. Instance throttled to baseline performance."
  resolution: "Enable T2 unlimited or upgrade to T3/larger instance type."

- symptoms: "High CPU with high steal time (st > 10%)"
  diagnosis: "Noisy neighbor on shared tenancy host."
  resolution: "Stop+start to migrate to different host. Consider dedicated tenancy for consistent performance."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "CloudWatch get-metric-statistics for CPUUtilization: GREEN — read-only"
  - "top/ps via SSM to identify processes: GREEN — read-only diagnostics"
  - "Kill runaway process via SSM: YELLOW — process terminated, may restart via service manager"
  - "Enable T2/T3 unlimited mode: YELLOW — may incur charges, recoverable by switching back"
  - "Stop+start to migrate host (noisy neighbor): YELLOW — changes public IP, brief downtime"
  - "Upgrade instance type: YELLOW — requires stop+start, recoverable by reverting type"
```

## Escalation Conditions
- Instance is in a production Auto Scaling group and cannot be stopped for resizing
- CPU exhaustion is caused by a suspected security incident (cryptominer, compromised process)
- High steal time persists after stop+start (potential platform issue)
- Multiple instances simultaneously experiencing CPU exhaustion (possible DDoS or cascading failure)
- Killing the CPU-consuming process would impact production service availability

## Data Sensitivity
- HIGH: get-console-output (may contain application logs with credentials or PII)
- HIGH: SSM command output from top/ps (reveals running processes, usernames, command arguments)
- MEDIUM: CloudWatch CPUUtilization metrics (reveals workload patterns and capacity)

## Prohibited Actions
- NEVER suggest terminating an instance to fix CPU exhaustion — use stop+start instead
- NEVER suggest killing processes without first identifying what they are
- NEVER suggest disabling CPU throttling on T-series instances without explaining cost implications
- NEVER suggest changing to dedicated tenancy without understanding pricing impact

## Phase 3 — Rollback
- If instance type was upgraded: stop instance, change back to original type, restart
- If T2 unlimited was enabled: revert with `modify-instance-credit-specification --cpu-credits standard`
- If process was killed: restart the process/service if it was legitimate
- If instance was stopped+started for host migration: no rollback needed (new host assignment is permanent)

## Output Format

```yaml
root_cause: "<process_runaway|credit_exhaustion|undersized|noisy_neighbor> — <detail>"
evidence:
  - type: cloudwatch_metric
    content: "<CPUUtilization data>"
severity: HIGH
mitigation:
  immediate: "Kill runaway process or upgrade instance"
  long_term: "Right-size instance, optimize application, enable auto-scaling"
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
