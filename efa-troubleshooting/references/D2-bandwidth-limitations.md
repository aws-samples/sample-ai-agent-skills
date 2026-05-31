---
title: "D2 — Bandwidth Limitations"
description: "Diagnose EFA bandwidth limitations and throughput bottlenecks"
status: active
severity: MEDIUM
triggers: ["bandwidth limited", "throughput bottleneck", "not reaching line rate", "efa bandwidth cap"]
owner: devops-agent
objective: "Identify and resolve EFA bandwidth limitations to maximize throughput"
context: "EFA bandwidth varies by instance type and number of interfaces. Achieving maximum bandwidth requires proper configuration of all interfaces and application tuning."
---

## Phase 1 — Triage

MUST:
- Check instance type bandwidth limits: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.{Bandwidth:NetworkPerformance,MaxEfa:EfaInfo.MaximumEfaInterfaces}'`
- Count active EFA interfaces: `aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=<instance-id>" "Name=interface-type,Values=efa" --query 'NetworkInterfaces | length(@)'`
- Measure current bandwidth: `ib_send_bw -d efa_0 --report_gbits -s 65536`

SHOULD:
- Test bandwidth on each EFA interface individually
- Check for network throttling in CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkPacketsOut --dimensions Name=InstanceId,Value=<id> --start-time <start> --end-time <end> --period 60 --statistics Sum`

MAY:
- Profile per-interface utilization during workload execution
- Compare single-stream vs multi-stream bandwidth

## Phase 2 — Remediate

MUST:
- Ensure all available EFA interfaces are attached and configured
- Verify application uses all EFA interfaces (not just the first one)
- Confirm instances are in a cluster placement group

SHOULD:
- For NCCL: set `NCCL_MIN_NCHANNELS` and `NCCL_MAX_NCHANNELS` to utilize all interfaces
- Configure multi-rail for MPI: `--mca btl_ofi_num_cqe_per_comp 1`

MAY:
- Tune TCP buffer sizes for hybrid TCP/EFA workloads
- Use DPDK for custom high-throughput applications

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-instance-types: GREEN — read-only specification query"
  - "ib_send_bw: GREEN — read-only bandwidth measurement"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric query"
  - "NCCL channel tuning: YELLOW — may affect training stability"
```

## Escalation Conditions

- Bandwidth is below 50% of published specifications with correct configuration
- Individual EFA interfaces show different bandwidth capabilities
- Bandwidth degrades during long-running workloads
- AWS network throttling is detected in CloudWatch metrics

## Data Sensitivity

- Instance type specifications: LOW — public information
- Bandwidth measurements: LOW — performance numbers
- CloudWatch metrics: LOW — numeric network statistics
- Interface configuration: LOW — network interface metadata

## Prohibited Actions

- NEVER assume all EFA interfaces provide equal bandwidth without testing
- NEVER increase NCCL channels beyond the number of EFA interfaces
- NEVER disable flow control to increase bandwidth — it causes packet loss
- NEVER compare bandwidth across different instance types as a diagnostic

## Phase 3 — Rollback

1. Revert NCCL channel configuration to defaults
2. Restore MPI multi-rail settings to previous values
3. If additional EFA interfaces were attached, detach if not needed
4. Re-run bandwidth tests to confirm baseline
5. Document the bandwidth limitations and instance type constraints

## Common Issues

- symptoms: "Only achieving 100 Gbps on p4d.24xlarge despite 400 Gbps capability"
  diagnosis: "Application is only using one of four EFA interfaces."
  resolution: "Configure application to use all 4 EFA interfaces. For NCCL, set NCCL_MIN_NCHANNELS=4."

- symptoms: "Bandwidth drops significantly with small message sizes"
  diagnosis: "Small messages are latency-bound, not bandwidth-bound. This is expected behavior."
  resolution: "Use message aggregation or batching for small messages. Focus on latency optimization instead."

## Output Format

```yaml
root_cause: "bandwidth — <specific_cause>"
evidence:
  - type: bandwidth_metric
    content: "<metric details>"
severity: MEDIUM
mitigation:
  immediate: "Configure multi-interface utilization"
  long_term: "Optimize application communication patterns"
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
