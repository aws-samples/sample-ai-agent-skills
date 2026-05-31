---
name: outposts-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Outposts problems
  by analyzing rack connectivity, compute instances, local storage,
  networking, management operations, hybrid connectivity, and security
  configurations using structured runbooks.
  Activate when: Outpost connectivity issues, capacity problems, hardware
  failures, EC2 on Outposts errors, ECS on Outposts failures, EBS local
  volume issues, S3 on Outposts problems, local gateway configuration,
  service link failures, update failures, monitoring gaps, region
  connectivity loss, failover issues, encryption problems, IAM on
  Outposts errors, or the user says something is wrong with Outposts
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Outposts, EC2, ECS, EBS, S3,
  CloudWatch, CloudTrail, IAM, and VPC permissions. Physical Outpost
  rack must be installed and connected.
---

# Outposts Diagnostics

## When to use

Any AWS Outposts investigation where the console alone is insufficient — rack connectivity analysis, compute troubleshooting, local storage debugging, networking issues, management operations, hybrid connectivity, or security configuration problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws outposts list-outposts
aws outposts get-outpost --outpost-id <outpost-id>
aws outposts get-outpost-instance-types --outpost-id <outpost-id>
aws ec2 describe-instances --filters "Name=outpost-arn,Values=<outpost-arn>"
aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name InstanceFamilyCapacityAvailability --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average
```

### Step 2 — Domain deep dive

```
aws outposts list-assets --outpost-identifier <outpost-id>
aws ec2 describe-local-gateway-route-tables
aws ec2 describe-local-gateway-virtual-interfaces
aws ec2 describe-local-gateway-route-table-vpc-associations
aws s3control list-regional-buckets --account-id <account-id> --outpost-id <outpost-id>
aws ec2 describe-coip-pools
aws ec2 describe-local-gateway-route-table-virtual-interface-group-associations
```

### Step 3 — Detailed investigation

```
aws outposts list-catalog-items
aws ec2 describe-subnets --filters "Name=outpost-arn,Values=<outpost-arn>"
aws ec2 describe-volumes --filters "Name=outpost-arn,Values=<outpost-arn>"
aws ecs list-clusters
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com
aws ec2 describe-network-interfaces --filters "Name=outpost-arn,Values=<outpost-arn>"
```

Read `references/guardrails.md` before concluding on any Outposts issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `outposts get-outpost` | Outpost details and status |
| `outposts list-assets` | Hardware asset inventory |
| `outposts get-outpost-instance-types` | Available instance types |
| `ec2 describe-instances` | EC2 instances on Outpost |
| `ec2 describe-local-gateway-route-tables` | Local gateway routing |
| `ec2 describe-coip-pools` | Customer-owned IP pools |
| `s3control list-regional-buckets` | S3 on Outposts buckets |
| `ec2 describe-volumes` | EBS volumes on Outpost |
| `outposts list-catalog-items` | Available configurations |
| `cloudwatch get-metric-statistics` | Outpost metrics |

## Gotchas: Outposts

- Service link: The service link connects the Outpost to the parent AWS Region. If the service link goes down, you can still use local instances and storage, but cannot launch new instances or make API calls that require the Region.
- Capacity is finite: Unlike the Region, Outpost capacity is physically limited. Instance launches fail with InsufficientInstanceCapacity when the rack is full. Monitor capacity proactively.
- S3 on Outposts: Uses S3 access points exclusively — you cannot use bucket URLs directly. Data stays local on the Outpost. Bucket naming follows s3-outposts ARN format.
- Local gateway: Required for on-premises network connectivity. Supports BGP for dynamic routing. Customer-owned IP (CoIP) pools provide addresses routable on the local network.
- EBS local: EBS volumes on Outposts are stored locally on the rack. Snapshots are stored in the parent Region S3. Volume types available depend on Outpost configuration.
- Instance types: Only specific instance families are available on Outposts (c5, m5, r5, g4dn, i3en, etc.). Not all Region instance types are supported.
- Networking: Outpost subnets are in the same VPC as Region subnets. Security groups and NACLs work the same way. Route tables can route to the local gateway for on-premises traffic.
- Updates: AWS manages firmware and software updates. Updates require service link connectivity. Some updates may require brief instance reboots.
- Shared Outposts: An Outpost can be shared with other AWS accounts via RAM. The owner account manages the physical infrastructure.
- DNS: Outpost instances use the VPC DNS resolver in the Region by default. If the service link is down, DNS resolution may fail unless local DNS is configured.

## Anti-hallucination rules

1. Always verify Outpost connectivity status before diagnosing compute or storage issues.
2. Never claim unlimited capacity on Outposts — capacity is physically constrained by the rack hardware.
3. S3 on Outposts requires access points — never suggest using standard S3 bucket URLs.
4. Service link outages do NOT stop running instances — only new launches and Region API calls are affected.
5. Local gateway configuration is required for on-premises connectivity — it is not automatic.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Rack | A1-A3 | Connectivity, capacity, hardware failures |
| B — Compute | B1-B2 | EC2 on Outposts, ECS on Outposts |
| C — Storage | C1-C2 | EBS local, S3 on Outposts |
| D — Networking | D1-D2 | Local gateway, service link |
| E — Management | E1-E2 | Updates, monitoring |
| F — Hybrid | F1-F2 | Region connectivity, failover |
| G — Security | G1-G2 | Encryption, IAM |
| Z — Catch-All | Z1 | General troubleshooting |
