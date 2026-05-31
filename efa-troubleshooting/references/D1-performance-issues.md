---
title: "D1 — Performance Issues"
description: "Diagnose EFA performance degradation and latency problems"
status: active
severity: HIGH
triggers: ["efa slow", "high latency", "low throughput", "performance degradation", "efa performance"]
owner: devops-agent
objective: "Identify and resolve EFA performance issues to achieve expected bandwidth and latency"
context: "EFA performance depends on instance type, placement group, driver version, message size, and application configuration."
---

## Phase 1 — Triage

MUST:
- Run EFA bandwidth test: `fi_pingpong -p efa -e rdm <remote-ip>` (measures latency and bandwidth)
- Verify placement group: `aws ec2 describe-instances --instance-ids <id1> <id2> --query 'Reservations[*].Instances[*].Placement.GroupName'`
- Check instance type bandwidth: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.{Bandwidth:NetworkPerformance,EfaSupported:EfaSupported}'`

SHOULD:
- Run comprehensive bandwidth test: `ib_send_bw -d efa_0 --report_gbits` (server) and `ib_send_bw -d efa_0 <server-ip> --report_gbits` (client)
- Check if hugepages are configured: `cat /proc/meminfo | grep HugePages`
- Verify all EFA interfaces are being utilized on multi-EFA instances

MAY:
- Profile application communication patterns to identify bottlenecks
- Compare performance with TCP baseline to quantify EFA benefit

## Phase 2 — Remediate

MUST:
- Ensure instances are in a cluster placement group
- Verify EFA driver and libfabric are up to date
- Configure hugepages: `echo 2048 | sudo tee /proc/sys/vm/nr_hugepages`

SHOULD:
- For multi-EFA instances, configure application to use all interfaces
- Tune memory registration limits: `ulimit -l unlimited`
- Set CPU affinity for network-intensive processes

MAY:
- Tune EFA-specific parameters: `export FI_EFA_TX_SIZE=8192` and `export FI_EFA_RX_SIZE=8192`

## Safety Ratings

```
safety_ratings:
  - "fi_pingpong: GREEN — read-only performance test"
  - "ec2 describe-instances: GREEN — read-only instance query"
  - "echo to nr_hugepages: YELLOW — changes memory allocation, may affect other processes"
  - "ulimit -l unlimited: YELLOW — changes resource limits"
```

## Escalation Conditions

- Performance is significantly below published specifications for the instance type
- Latency spikes occur intermittently despite correct configuration
- Multi-EFA interface utilization is unbalanced
- Performance degrades over time during long-running jobs

## Data Sensitivity

- Performance test results: LOW — bandwidth and latency numbers
- Placement group information: LOW — group names and strategies
- Instance type specifications: LOW — public AWS documentation
- Hugepage configuration: LOW — memory allocation settings

## Prohibited Actions

- NEVER allocate all system memory to hugepages — leave sufficient for OS and applications
- NEVER change CPU affinity on production workloads without testing
- NEVER compare EFA performance across different instance types as equivalent
- NEVER ignore placement group configuration when diagnosing performance

## Phase 3 — Rollback

1. Revert hugepage configuration: `echo 0 | sudo tee /proc/sys/vm/nr_hugepages`
2. Restore ulimit to original values
3. Revert any EFA tuning environment variables
4. If placement group was changed, migrate instances back
5. Re-run performance tests to confirm baseline is restored

## Common Issues

- symptoms: "EFA bandwidth is 10x lower than expected for the instance type"
  diagnosis: "Instances are not in a cluster placement group, causing cross-rack communication."
  resolution: "Create a cluster placement group and relaunch instances within it."

- symptoms: "Small message latency is high despite EFA being configured"
  diagnosis: "Hugepages are not configured, causing memory registration overhead."
  resolution: "Configure hugepages: echo 2048 | sudo tee /proc/sys/vm/nr_hugepages."

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: performance_metric
    content: "<metric details>"
severity: HIGH
mitigation:
  immediate: "Fix placement group or driver configuration"
  long_term: "Standardize EFA performance tuning in launch templates"
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
