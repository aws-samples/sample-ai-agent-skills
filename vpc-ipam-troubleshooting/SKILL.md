---
name: vpc-ipam-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon VPC IPAM
  problems by analyzing IPAM creation, scope configuration, pool management,
  CIDR provisioning, allocation failures, overlap detection, organization
  integration, cross-region configuration, BYOIP, public IP insights,
  compliance monitoring, and following structured runbooks.
  Activate when: IPAM creation failures, scope configuration issues, pool
  errors, CIDR provisioning problems, allocation failures, overlap detection,
  organization integration issues, cross-region problems, BYOIP provisioning,
  public IP insights, compliance monitoring, or the user says something is
  wrong with VPC IPAM.
compatibility: >
  Requires AWS CLI or SDK access with ec2 (IPAM), organizations, ram,
  iam, cloudtrail, and cloudwatch permissions.
---

# VPC IPAM Diagnostics

## When to use

Any VPC IPAM investigation — IPAM creation, scope configuration, pool management, CIDR provisioning, allocation failures, overlap detection, organization integration, cross-region configuration, BYOIP, public IP insights, or compliance monitoring.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-ipams --query 'Ipams[*].{IpamId:IpamId,State:State,ScopeCount:ScopeCount,OperatingRegions:OperatingRegions}'
aws ec2 describe-ipam-scopes --query 'IpamScopes[*].{ScopeId:IpamScopeId,Type:IpamScopeType,PoolCount:PoolCount}'
aws ec2 describe-ipam-pools --query 'IpamPools[*].{PoolId:IpamPoolId,State:State,AddressFamily:AddressFamily,Locale:Locale}'
```

### Step 2 — Domain deep dive

```
aws ec2 get-ipam-pool-cidrs --ipam-pool-id <pool-id> --query 'IpamPoolCidrs[*].{Cidr:Cidr,State:State}'
aws ec2 get-ipam-pool-allocations --ipam-pool-id <pool-id> --query 'IpamPoolAllocations[*].{Cidr:Cidr,ResourceId:ResourceId,ResourceType:ResourceType}'
aws ec2 get-ipam-resource-cidrs --ipam-scope-id <scope-id> --query 'IpamResourceCidrs[*].{ResourceId:ResourceId,ResourceCidr:ResourceCidr,ComplianceStatus:ComplianceStatus}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
aws ec2 get-ipam-discovered-resource-cidrs --ipam-resource-discovery-id <discovery-id> --resource-region <region>
aws ec2 describe-ipam-resource-discoveries --query 'IpamResourceDiscoveries[*].{DiscoveryId:IpamResourceDiscoveryId,State:State,IsDefault:IsDefault}'
```

Read `references/guardrails.md` before concluding on any VPC IPAM issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ec2 describe-ipams` | List IPAMs |
| `ec2 describe-ipam-scopes` | List IPAM scopes |
| `ec2 describe-ipam-pools` | List IPAM pools |
| `ec2 get-ipam-pool-cidrs` | Check pool CIDR provisioning |
| `ec2 get-ipam-pool-allocations` | Check pool allocations |
| `ec2 get-ipam-resource-cidrs` | Check resource CIDR compliance |
| `ec2 get-ipam-discovered-resource-cidrs` | View discovered resources |

## Gotchas: VPC IPAM

- IPAM operates at the organization level. A single IPAM can manage IP addresses across all accounts and regions in an AWS Organization.
- IPAM has two default scopes: public and private. Custom scopes can be created for additional isolation. Pools belong to scopes.
- Pool locale determines which region can allocate from the pool. A pool with locale us-east-1 cannot allocate CIDRs for VPCs in eu-west-1.
- CIDR provisioning to pools is asynchronous. After provisioning, the CIDR state transitions from pending-provision to provisioned. Do not allocate until provisioned.
- Allocation failures often indicate the pool has insufficient free space. Check pool utilization before diagnosing complex issues.
- IPAM resource discovery is required for cross-account visibility. Without it, IPAM cannot see resources in member accounts.
- BYOIP (Bring Your Own IP) requires address range verification with Regional Internet Registries before provisioning to IPAM.

## Anti-hallucination rules

1. Always cite specific IPAM IDs, pool IDs, CIDR ranges, or API responses as evidence.
2. Pool locale restricts allocation to a specific region. Never suggest allocating from a pool with the wrong locale.
3. CIDR provisioning is asynchronous. Never assume a CIDR is available immediately after provisioning.
4. IPAM resource discovery is required for cross-account visibility. Never assume IPAM sees all accounts automatically.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — IPAM Setup | A1–A2 | IPAM creation failures, scope configuration |
| B — Pools | B1–B2 | Pool creation errors, CIDR provisioning |
| C — Allocations | C1–C2 | Allocation failures, CIDR overlap detection |
| D — Organization | D1–D2 | Organization integration, cross-region configuration |
| E — BYOIP & Public IP | E1–E2 | BYOIP provisioning, public IP insights |
| F — Compliance | F1 | Compliance monitoring |
| Z — Catch-All | Z1 | General troubleshooting |
