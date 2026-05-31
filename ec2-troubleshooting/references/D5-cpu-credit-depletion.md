---
title: "D5 — T-Series CPU Credit Depletion"
description: "Diagnose performance degradation on T-series instances due to CPU credit exhaustion"
status: active
severity: MEDIUM
triggers:
  - "CPUCreditBalance.*0"
  - "CPUSurplusCreditsCharged"
  - "T2.*slow"
  - "T3.*throttled"
owner: devops-agent
objective: "Restore CPU performance by managing credits or changing instance configuration"
context: "T-series instances (T2, T3, T3a, T4g) use CPU credits for bursting above baseline. T2 default: standard mode (hard throttle at 0 credits). T3/T3a/T4g default: unlimited mode (can go negative, charged per vCPU-hour). Baseline varies by instance size."
---

## Phase 1 — Triage

MUST:
- Check CPUCreditBalance: `aws cloudwatch get-metric-statistics --metric-name CPUCreditBalance`
- Check CPUCreditUsage: how fast credits are being consumed
- Check CPUSurplusCreditBalance (unlimited mode): negative credit balance
- Check CPUSurplusCreditsCharged: actual charges for surplus usage
- Determine if instance is in standard or unlimited mode

SHOULD:
- Check baseline CPU percentage for the instance size (e.g., t3.micro = 10%, t3.small = 20%)
- Review CPUUtilization pattern — sustained above baseline depletes credits

MAY:
- Calculate credit earn rate vs usage rate to predict depletion

## Phase 2 — Remediate

MUST:
- If T2 standard with 0 credits: enable unlimited mode or upgrade instance type
- If T3 unlimited with high surplus charges: workload exceeds burstable model, switch to fixed-performance (M/C series)
- If temporary spike: wait for credits to accumulate (earned at baseline rate)

SHOULD:
- Set CloudWatch alarm on CPUCreditBalance < threshold
- Evaluate if workload is truly burstable or sustained

## Common Issues

- symptoms: "T2 instance suddenly slow, CPUCreditBalance = 0"
  diagnosis: "T2 standard mode, credits exhausted. CPU throttled to baseline."
  resolution: "Enable T2 unlimited: `aws ec2 modify-instance-credit-specification --instance-credit-specifications InstanceId=<id>,CpuCredits=unlimited`"

- symptoms: "T3 unlimited with high CPUSurplusCreditsCharged"
  diagnosis: "Workload consistently exceeds baseline. Unlimited mode is expensive for sustained loads."
  resolution: "Switch to fixed-performance instance (m5/m6i) — often cheaper than T3 unlimited for sustained workloads."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "CloudWatch CPUCreditBalance, CPUCreditUsage metrics: GREEN — read-only"
  - "describe-instance-credit-specifications: GREEN — read-only"
  - "Enable unlimited mode: YELLOW — may incur surplus charges, recoverable by switching back"
  - "Upgrade to fixed-performance instance type: YELLOW — requires stop+start, recoverable"
  - "Modify instance credit specification to standard: YELLOW — may cause throttling, recoverable"
```

## Escalation Conditions
- T-series instance is running a production workload that consistently exceeds baseline
- Surplus credit charges are accumulating significantly (unlimited mode cost concern)
- Instance cannot be stopped for type change due to production constraints
- Credit depletion is affecting an Auto Scaling group with multiple T-series instances
- Workload pattern is unclear (cannot determine if burstable or sustained)

## Data Sensitivity
- MEDIUM: CloudWatch CPU credit metrics (reveals workload patterns and instance utilization)
- MEDIUM: describe-instance-credit-specifications (reveals credit mode configuration)
- LOW: describe-instance-types (public baseline performance data)

## Prohibited Actions
- NEVER suggest switching from unlimited to standard mode on a production instance without warning about throttling
- NEVER suggest T2 instances for sustained workloads (T3/T3a have better baseline and unlimited defaults)
- NEVER suggest disabling CloudWatch alarms on credit balance to hide the problem
- NEVER suggest oversizing to the largest T-series instance when a fixed-performance type would be more cost-effective

## Phase 3 — Rollback
- If unlimited mode was enabled: revert with `modify-instance-credit-specification --cpu-credits standard`
- If instance type was changed to fixed-performance: stop, change back to T-series type, restart
- If instance type was upgraded within T-series: stop, change back to original size, restart
- If CloudWatch alarm was set: delete alarm if no longer needed with `delete-alarms`

## Output Format

```yaml
root_cause: "<credit_exhaustion|sustained_above_baseline|wrong_instance_class> — <detail>"
evidence:
  - type: cloudwatch_metric
    content: "<CPUCreditBalance and CPUUtilization data>"
severity: MEDIUM
mitigation:
  immediate: "Enable unlimited mode or upgrade instance"
  long_term: "Right-size to fixed-performance instance if workload is sustained"
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
