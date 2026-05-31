---
name: localzones-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Local Zones
  problems by analyzing zone enablement, subnet creation, instance
  launches, EBS volumes, networking, ELB integration, RDS, latency
  testing, capacity planning, VPC extension, and following structured
  runbooks.
  Activate when: Local Zone enablement failures, subnet creation errors,
  instance launch issues, instance type availability problems, EBS
  volume issues, networking configuration errors, ELB integration
  problems, RDS Local Zone issues, latency testing failures, capacity
  planning concerns, VPC extension issues, or the user says something
  is wrong with Local Zones.
compatibility: >
  Requires AWS CLI or SDK access with ec2, elasticloadbalancing, rds,
  cloudwatch, and iam permissions.
---

# AWS Local Zones Diagnostics

## When to use

Any AWS Local Zones investigation — zone enablement, subnet creation, instance launches, instance type availability, EBS volumes, networking, ELB integration, RDS, latency testing, capacity planning, or VPC extension.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-availability-zones --filters "Name=zone-type,Values=local-zone" --query 'AvailabilityZones[*].{ZoneName:ZoneName,State:State,OptInStatus:OptInStatus,GroupName:GroupName}'
aws ec2 describe-subnets --filters "Name=availability-zone,Values=<local-zone-name>" --query 'Subnets[*].{SubnetId:SubnetId,VpcId:VpcId,CidrBlock:CidrBlock,State:State}'
aws ec2 describe-instances --filters "Name=availability-zone,Values=<local-zone-name>" --query 'Reservations[*].Instances[*].{InstanceId:InstanceId,InstanceType:InstanceType,State:State.Name}'
```

### Step 2 — Domain deep dive

```
aws ec2 describe-instance-type-offerings --location-type availability-zone --filters "Name=location,Values=<local-zone-name>" --query 'InstanceTypeOfferings[*].InstanceType'
aws ec2 describe-volumes --filters "Name=availability-zone,Values=<local-zone-name>" --query 'Volumes[*].{VolumeId:VolumeId,VolumeType:VolumeType,State:State,Size:Size}'
aws ec2 describe-network-interfaces --filters "Name=availability-zone,Values=<local-zone-name>"
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkIn --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"
```

Read `references/guardrails.md` before concluding on any Local Zones issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ec2 describe-availability-zones` | List Local Zones and opt-in status |
| `ec2 describe-instance-type-offerings` | Check available instance types in zone |
| `ec2 describe-subnets` | List subnets in Local Zone |
| `ec2 describe-instances` | List instances in Local Zone |
| `ec2 describe-volumes` | List EBS volumes in Local Zone |
| `ec2 describe-network-interfaces` | List ENIs in Local Zone |
| `ec2 describe-route-tables` | Check routing for Local Zone subnets |

## Gotchas: AWS Local Zones

- Local Zones require explicit opt-in. They are NOT enabled by default. Use `aws ec2 modify-availability-zone-group --group-name <group> --opt-in-status opted-in` to enable.
- Not all instance types are available in every Local Zone. Available types vary significantly by zone. Always check offerings before launching.
- EBS volume types are limited in Local Zones. Not all volume types (io2, sc1, st1) are available. gp2 and gp3 are most commonly supported.
- Local Zone subnets are part of the parent region VPC but traffic between Local Zone and parent region traverses the internet or Direct Connect.
- Network ACLs and security groups work the same as in the parent region, but internet gateways are shared with the parent region VPC.
- RDS support in Local Zones is limited to specific DB engines and instance classes. Multi-AZ is NOT supported in Local Zones.
- ELB in Local Zones requires subnets in the Local Zone. Not all ELB types may be available in every Local Zone.

## Anti-hallucination rules

1. Always cite specific zone names, instance type offerings, or API responses as evidence.
2. Local Zones and Wavelength Zones are different. Never conflate them.
3. Not all AWS services are available in Local Zones. Never assume service availability without checking.
4. Local Zone pricing may differ from parent region. Never assume identical pricing.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Enablement | A1-A2 | Zone enablement, subnet creation |
| B — Instances | B1-B2 | Launch errors, type availability |
| C — Storage & Network | C1-C2 | EBS volumes, networking configuration |
| D — Integration | D1-D2 | ELB integration, RDS Local Zone |
| E — Performance | E1-E2 | Latency testing, capacity planning |
| F — VPC | F1 | VPC extension issues |
| Z — Catch-All | Z1 | General troubleshooting |
