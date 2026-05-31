---
title: "D4 — NAT Gateway Bandwidth Limits"
description: "Diagnose throughput issues caused by NAT gateway bandwidth limits"
status: active
severity: MEDIUM
triggers:
  - "Slow transfers through NAT"
  - "Throughput degradation"
  - "NAT gateway bandwidth"
owner: devops-agent
objective: "Identify and resolve NAT gateway bandwidth bottlenecks"
context: "NAT gateway supports up to 100 Gbps bandwidth (bursts to 100 Gbps, sustained ~45 Gbps). Single flow limited to ~5 Gbps. If you need more, split across multiple NAT gateways."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch metrics: `BytesOutToDestination`, `BytesOutToSource`, `BytesInFromDestination`, `BytesInFromSource`
- Calculate throughput and compare to NAT gateway limits
- Check if traffic is concentrated in a single flow (5 Gbps per flow limit)

SHOULD:
- Check if multiple AZs share a single NAT gateway (cross-AZ + bandwidth issue)
- Verify instance network bandwidth isn't the bottleneck instead
- Check PacketsDropCount for drops due to bandwidth

MAY:
- Analyze flow logs to identify high-bandwidth flows
- Consider if the workload should use VPC endpoints instead of NAT

## Common Issues

- symptoms: "Large data transfers through NAT are slow"
  diagnosis: "Single flow limited to ~5 Gbps through NAT gateway."
  resolution: "Parallelize transfers across multiple connections/flows, or use VPC endpoints."

- symptoms: "All AZs routing through one NAT gateway"
  diagnosis: "Single NAT gateway handling all private subnet traffic, cross-AZ data transfer costs apply."
  resolution: "Deploy one NAT gateway per AZ for bandwidth and cost optimization."

## Safety Ratings

```
safety_ratings:
  - "get-metric-statistics (BytesOutToDestination, etc.): GREEN — read-only CloudWatch metric query"
  - "describe-nat-gateways: GREEN — read-only NAT gateway inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "create-nat-gateway (add per-AZ NAT): YELLOW — creates new NAT gateway, recoverable by deleting"
  - "create-route (route AZ traffic to local NAT): YELLOW — adds route, recoverable by deleting"
  - "replace-route (redirect traffic to new NAT): YELLOW — modifies route target, recoverable by replacing back"
```

## Escalation Conditions

- "Fix requires creating additional NAT gateways in production VPC"
- "Fix requires modifying production VPC route tables"
- "Bandwidth bottleneck affecting multiple services"
- "Fix involves cross-AZ routing changes — affects cost and latency"

## Data Sensitivity

- HIGH: VPC flow logs (contain source/destination IPs and ports)
- HIGH: NAT gateway throughput metrics (expose traffic volume patterns)
- MEDIUM: CloudWatch metrics, NAT gateway configuration, AZ placement

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete an existing NAT gateway to resolve bandwidth issues"

## Phase 3 — Rollback

- If a new NAT gateway was created: delete it with `aws ec2 delete-nat-gateway --nat-gateway-id <id>` and release the EIP
- If routes were changed to per-AZ NAT: replace them back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <original-nat-id>`
- If a VPC endpoint was created to offload traffic: delete it with `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
- Document original NAT gateway configuration, route tables, and AZ assignments before changes

## Output Format

```yaml
root_cause: "nat_bandwidth — <detail>"
evidence:
  - type: cloudwatch_metric
    content: "<throughput metrics>"
severity: MEDIUM
mitigation:
  immediate: "Distribute traffic across multiple NAT gateways"
  long_term: "One NAT per AZ, use VPC endpoints for AWS services, parallelize large transfers"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
