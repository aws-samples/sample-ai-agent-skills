---
name: efa-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Elastic Fabric
  Adapter (EFA) problems by analyzing EFA creation, attachment,
  driver installation, libfabric, MPI communication, NCCL,
  performance, bandwidth, security groups, placement groups,
  SageMaker integration, and following structured runbooks.
  Activate when: EFA creation failures, attachment errors, driver
  issues, libfabric problems, MPI errors, NCCL failures,
  performance degradation, bandwidth limitations, security group
  misconfigurations, placement group issues, SageMaker EFA
  problems, or the user says something is wrong with EFA.
compatibility: >
  Requires AWS CLI or SDK access with ec2, ecs, eks, sagemaker,
  and cloudwatch permissions. SSH or SSM access to instances for
  driver-level diagnostics.
---

# Elastic Fabric Adapter (EFA) Diagnostics

## When to use

Any EFA investigation — creation failures, attachment errors, driver installation, libfabric issues, MPI communication errors, NCCL failures, performance problems, bandwidth limitations, security group configuration, placement groups, or SageMaker integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].NetworkInterfaces[*].{Id:NetworkInterfaceId,Type:InterfaceType,Status:Status}'
aws ec2 describe-network-interfaces --filters "Name=interface-type,Values=efa" --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,InstanceId:Attachment.InstanceId,Status:Status,AZ:AvailabilityZone}'
aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].{EfaSupported:NetworkInfo.EfaSupported,MaxEfa:NetworkInfo.EfaInfo.MaximumEfaInterfaces}'
```

### Step 2 — Domain deep dive

```
fi_info -p efa          # List EFA provider capabilities
fi_pingpong -p efa      # Test basic EFA connectivity between nodes
ibv_devinfo             # Show EFA device information and capabilities
```

### Step 3 — Detailed investigation

```
aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].{Inbound:IpPermissions,Outbound:IpPermissionsEgress}'
aws ec2 describe-placement-groups --group-names <group> --query 'PlacementGroups[0].{Strategy:Strategy,State:State,GroupId:GroupId}'
dmesg | grep -i efa     # Check kernel messages for EFA driver issues
```

Read `references/guardrails.md` before concluding on any EFA issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ec2 describe-network-interfaces` | List EFA interfaces and status |
| `ec2 describe-instance-types` | Check EFA support for instance type |
| `fi_info -p efa` | Verify libfabric EFA provider |
| `fi_pingpong -p efa` | Test EFA connectivity |
| `ibv_devinfo` | Check EFA device capabilities |
| `ec2 describe-security-groups` | Verify EFA security group rules |
| `ec2 describe-placement-groups` | Check placement group configuration |

## Gotchas: Elastic Fabric Adapter

- EFA is only supported on specific instance types (p4d, p5, trn1, hpc6a, hpc7g, c5n.18xlarge, etc.). Not all large instances support EFA.
- EFA requires a security group that allows ALL traffic from itself (self-referencing rule). Without this, EFA traffic is blocked.
- EFA uses OS-bypass for kernel-bypass networking. The EFA driver and libfabric must be installed on the instance — they are NOT included in standard AMIs.
- Placement groups with cluster strategy are STRONGLY recommended for EFA. Without them, instances may be placed across different network switches, degrading performance.
- EFA and ENA (Elastic Network Adapter) are different. EFA provides additional OS-bypass capabilities on top of ENA. An EFA interface also functions as an ENA interface.
- NCCL (NVIDIA Collective Communications Library) requires the aws-ofi-nccl plugin to use EFA. Without it, NCCL falls back to TCP sockets.
- EFA interfaces cannot be attached to running instances — they must be specified at launch time or attached when the instance is stopped.

## Anti-hallucination rules

1. Always cite specific instance types, interface IDs, or driver versions as evidence.
2. EFA support is instance-type specific. Never assume an instance type supports EFA without checking.
3. EFA requires OS-bypass drivers. Never suggest EFA works with standard networking drivers alone.
4. Security group self-referencing rules are mandatory. Never skip this requirement.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Creation & Attachment | A1–A2 | EFA creation failures, attachment errors |
| B — Drivers & Libraries | B1–B2 | Driver installation, libfabric issues |
| C — Communication | C1–C2 | MPI communication errors, NCCL errors |
| D — Performance | D1–D2 | Performance issues, bandwidth limitations |
| E — Network Config | E1–E2 | Security group configuration, placement groups |
| F — Integration | F1 | SageMaker integration |
| Z — Catch-All | Z1 | General troubleshooting |
