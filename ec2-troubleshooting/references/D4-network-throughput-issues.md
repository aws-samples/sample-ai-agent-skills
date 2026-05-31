---
title: "D4 — Network Throughput / Latency Issues"
description: "Diagnose network performance degradation on EC2 instances"
status: active
severity: MEDIUM
triggers:
  - "NetworkPacketsIn.*drop"
  - "network.*slow"
  - "packet loss"
  - "latency.*high"
  - "ENA.*throttle"
owner: devops-agent
objective: "Identify the network performance bottleneck and restore throughput"
context: "EC2 network performance depends on instance type (bandwidth allocation), ENA driver, placement (same AZ vs cross-AZ vs cross-region), and VPC configuration. Enhanced networking (ENA) is required for high performance. Network bandwidth is shared across all ENIs."
---

## Phase 1 — Triage

MUST:
- Check instance type network performance: `aws ec2 describe-instance-types --instance-types <type>` → NetworkInfo
- Check CloudWatch network metrics: NetworkIn, NetworkOut, NetworkPacketsIn, NetworkPacketsOut
- Check ENA metrics (Nitro instances): `ethtool -S <interface>` via SSM — look for bw_in_allowance_exceeded, pps_allowance_exceeded, conntrack_allowance_exceeded
- Verify enhanced networking is enabled: `aws ec2 describe-instance-attribute --instance-id <id> --attribute enaSupport`

SHOULD:
- Check if traffic is cross-AZ (adds latency and cost) or same-AZ
- Check for ENA driver version: `modinfo ena` via SSM
- Check MTU settings: `ip link show` via SSM

MAY:
- Run iperf3 between instances to measure actual throughput
- Check placement group membership for low-latency requirements

## Common Issues

- symptoms: "ENA metrics show bw_in_allowance_exceeded > 0"
  diagnosis: "Instance network bandwidth limit reached. Traffic is being shaped/dropped."
  resolution: "Upgrade to instance type with higher network bandwidth. Use placement groups for cluster traffic."

- symptoms: "High latency between instances"
  diagnosis: "Cross-AZ traffic (adds ~1ms), cross-region traffic, or network congestion."
  resolution: "Place communicating instances in same AZ. Use placement groups for low latency."

- symptoms: "Packet loss on small instance types"
  diagnosis: "Small instances have lower network allocation and burst credits. Sustained traffic exceeds allocation."
  resolution: "Upgrade instance type. Network performance scales with instance size."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instance-types, CloudWatch network metrics: GREEN — read-only"
  - "ethtool, ip link via SSM: GREEN — read-only diagnostics"
  - "Upgrade instance type for more bandwidth: YELLOW — requires stop+start, recoverable"
  - "Enable enhanced networking (ENA): YELLOW — requires stop+start, recoverable"
  - "Change placement group: YELLOW — requires stop+start, may fail if capacity unavailable"
  - "Modify MTU settings: YELLOW — may break connectivity if jumbo frames not supported end-to-end"
```

## Escalation Conditions
- Network throttling is causing production service degradation or timeouts
- Multiple instances in a placement group are experiencing bandwidth limits
- ENA driver issue requires kernel update on production instances
- Network performance issue is cross-AZ and requires architecture redesign
- Packet loss is intermittent and cannot be reproduced consistently

## Data Sensitivity
- HIGH: ethtool -S output (reveals detailed network statistics, interface names)
- MEDIUM: CloudWatch network metrics (reveals traffic patterns and bandwidth utilization)
- MEDIUM: iperf3 results (reveals actual network capacity between instances)
- LOW: describe-instance-types (public instance capability data)

## Prohibited Actions
- NEVER suggest disabling enhanced networking (ENA) to troubleshoot
- NEVER suggest enabling jumbo frames (MTU 9001) without verifying end-to-end support
- NEVER suggest disabling source/destination check without understanding the use case (NAT, routing)
- NEVER suggest moving instances out of a placement group without understanding latency requirements

## Phase 3 — Rollback
- If instance type was upgraded: stop instance, change back to original type, restart
- If MTU was changed: revert MTU with `ip link set dev <iface> mtu 1500` via SSM
- If placement group was changed: stop instance, move back to original placement group
- If ENA driver was updated: revert to previous driver version via SSM or rescue instance

## Output Format

```yaml
root_cause: "<bandwidth_limit|pps_limit|cross_az|ena_driver|mtu> — <detail>"
evidence:
  - type: ena_metrics
    content: "<ENA counter values showing throttling>"
severity: MEDIUM
mitigation:
  immediate: "Upgrade instance type or optimize traffic patterns"
  long_term: "Use placement groups, same-AZ deployment, ENA driver updates"
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
