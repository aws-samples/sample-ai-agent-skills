---
name: wavelength-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Wavelength problems
  by analyzing Wavelength Zone deployments, carrier gateway configurations,
  compute instances, networking, performance, and Region integration
  using structured runbooks.
  Activate when: Wavelength Zone issues, carrier gateway problems, EC2
  in Wavelength failures, ECS/EKS on Wavelength errors, carrier IP
  allocation issues, VPC subnet configuration, latency problems,
  throughput issues, Region connectivity, S3 access from Wavelength,
  or the user says something is wrong with Wavelength without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with EC2, ECS, EKS, VPC, CloudWatch,
  CloudTrail, and IAM permissions. Wavelength Zone must be enabled
  in the account.
---

# Wavelength Diagnostics

## When to use

Any AWS Wavelength investigation where the console alone is insufficient — Wavelength Zone deployment, carrier gateway configuration, compute troubleshooting, networking issues, ultra-low latency optimization, or Region integration problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-availability-zones --filters "Name=zone-type,Values=wavelength-zone" --all-availability-zones
aws ec2 describe-subnets --filters "Name=availability-zone,Values=<wavelength-zone>"
aws ec2 describe-instances --filters "Name=availability-zone,Values=<wavelength-zone>"
aws ec2 describe-carrier-gateways
aws ec2 describe-addresses --filters "Name=domain,Values=vpc" --query 'Addresses[?CarrierIp!=null]'
```

### Step 2 — Domain deep dive

```
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<wavelength-subnet-id>"
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>"
aws ec2 describe-network-interfaces --filters "Name=availability-zone,Values=<wavelength-zone>"
aws ecs list-clusters
aws eks list-clusters
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkIn --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>"
```

Read `references/guardrails.md` before concluding on any Wavelength issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-availability-zones` | List Wavelength Zones |
| `describe-carrier-gateways` | Carrier gateway status |
| `describe-addresses` | Carrier IP allocations |
| `describe-subnets` | Wavelength subnet config |
| `describe-instances` | EC2 in Wavelength |
| `describe-route-tables` | Routing configuration |
| `describe-network-interfaces` | ENI details |
| `describe-vpc-endpoints` | VPC endpoint access |

## Gotchas: Wavelength

- Wavelength Zones must be explicitly opted-in per account. They are not enabled by default.
- Carrier gateway is required for carrier network (mobile) traffic. It is NOT the same as an internet gateway or NAT gateway.
- Carrier IPs are allocated from the carrier's IP pool, not AWS. They are routable on the carrier's 5G network.
- Not all instance types are available in Wavelength Zones. Typically t3, r5, g4dn families.
- EBS volumes in Wavelength Zones are stored locally. Snapshots go to the parent Region.
- No NAT gateway or internet gateway support in Wavelength Zones. Use carrier gateway for mobile traffic or route through the Region for internet.
- S3 access from Wavelength requires routing through the Region VPC or using S3 gateway endpoints.
- Wavelength subnets are in the same VPC as Region subnets but in a different AZ type.
- DNS resolution uses the VPC resolver in the parent Region.
- Auto Scaling groups can span Wavelength Zones but require Wavelength-specific launch templates.

## Anti-hallucination rules

1. Always verify the Wavelength Zone is opted-in before diagnosing deployment issues.
2. Never suggest using internet gateways or NAT gateways in Wavelength Zones — they are not supported.
3. Carrier IPs are different from Elastic IPs — do not confuse them.
4. Not all services are available in Wavelength Zones — verify service availability before recommending.
5. Latency measurements must account for the carrier network path, not just AWS infrastructure.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Zone | A1-A2 | Wavelength zone, carrier gateway |
| B — Compute | B1-B2 | EC2 in Wavelength, ECS/EKS |
| C — Networking | C1-C2 | Carrier IP, VPC subnets |
| D — Performance | D1-D2 | Latency, throughput |
| E — Integration | E1-E2 | Region connectivity, S3 access |
| Z — Catch-All | Z1 | General troubleshooting |
