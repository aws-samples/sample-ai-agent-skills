---
name: outposts-servers-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Outposts Servers
  (1U/2U form factor) problems by analyzing server ordering, activation,
  EC2 launch errors, EBS local volumes, network connectivity, service link,
  capacity management, software updates, monitoring, tagging, and
  rack-vs-server differences using structured runbooks.
  Activate when: server ordering issues, activation failures, EC2 launch
  errors on Outposts server, EBS local volume problems, network connectivity
  issues, service link failures, capacity constraints, update failures,
  monitoring gaps, or the user says something is wrong with Outposts Servers.
compatibility: >
  Requires AWS CLI or SDK access with outposts, ec2, ebs, cloudwatch,
  ssm, iam, and networking permissions. Physical access to edge site
  may be required for hardware issues.
---

# AWS Outposts Servers Diagnostics

## When to use

Any AWS Outposts Servers (1U/2U) investigation — server ordering, activation, EC2 instance launches, EBS local volumes, network connectivity, service link, capacity management, software updates, monitoring, tagging, or rack-vs-server comparison questions.

## Investigation workflow

### Step 1 — Collect and triage

```
aws outposts list-outposts --query 'Outposts[*].{Id:OutpostId,Name:Name,Arn:OutpostArn,LifeCycleStatus:LifeCycleStatus,SiteId:SiteId}'
aws outposts get-outpost --outpost-id <outpost-id>
aws ec2 describe-instances --filters "Name=placement-group-name,Values=*" --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name,Placement:Placement}'
```

### Step 2 — Domain deep dive

```
aws outposts list-assets --outpost-identifier <outpost-id> --query 'Assets[*].{AssetId:AssetId,AssetType:AssetType,ComputeAttributes:ComputeAttributes}'
aws ec2 describe-instance-types --filters "Name=supported-usage-class,Values=outpost" --query 'InstanceTypes[*].{Type:InstanceType,VCpus:VCpuInfo.DefaultVCpus,Memory:MemoryInfo.SizeInMiB}'
aws outposts get-outpost-instance-types --outpost-id <outpost-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws ec2 describe-coip-pools --filters "Name=coip-pool.local-gateway-route-table-id,Values=<lgw-rtb-id>"
```

Read `references/guardrails.md` before concluding on any Outposts Servers issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `outposts list-outposts` | List all Outposts in the account |
| `outposts get-outpost` | Get details of a specific Outpost |
| `outposts list-assets` | List hardware assets on an Outpost |
| `outposts get-outpost-instance-types` | List supported instance types |
| `ec2 describe-instances` (with Outpost filter) | List EC2 instances on Outpost |
| `ec2 describe-coip-pools` | Check customer-owned IP pools |
| `cloudwatch get-metric-statistics` (AWS/Outposts) | Check connectivity metrics |

## Gotchas: AWS Outposts Servers

- Outposts Servers (1U/2U) are SINGLE-SERVER deployments, NOT rack-scale. They support a limited subset of instance types compared to Outposts racks.
- Outposts Servers require a service link back to the parent AWS Region. If the service link is down, new instance launches and API calls fail, but running instances continue.
- EBS local volumes on Outposts Servers are stored on the server's local NVMe storage. They do NOT replicate to the Region and are lost if the server hardware fails.
- Customer-owned IP (CoIP) pools are used for local network addressing. Misconfigured CoIP pools cause instance launch failures on Outposts.
- Outposts Servers support only specific instance families (c6gd, m6gd, r6gd for Graviton; c6id, m6id, r6id for Intel). Do not assume all EC2 instance types are available.
- Software updates on Outposts Servers are managed by AWS but require the service link to be active. Updates cannot be applied while disconnected.
- Outposts Servers have NO redundancy — a single hardware failure means all workloads on that server are affected.

## Anti-hallucination rules

1. Always cite specific Outpost IDs, asset IDs, or API responses as evidence.
2. Outposts Servers (1U/2U) and Outposts racks are DIFFERENT form factors. Never conflate capabilities.
3. EBS on Outposts Servers uses local NVMe — it does NOT replicate to the Region. Never claim regional durability.
4. Service link connectivity is required for control plane operations. Never claim full offline autonomy.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Ordering & Setup | A1–A2 | Server ordering issues, activation failures |
| B — Compute & Storage | B1–B2 | EC2 launch errors, EBS local volume issues |
| C — Networking | C1–C2 | Network connectivity, service link failures |
| D — Operations | D1–D2 | Capacity management, software updates |
| E — Observability | E1–E2 | Monitoring configuration, tag management |
| F — Architecture | F1 | Rack vs server differences |
| Z — Catch-All | Z1 | General troubleshooting |
