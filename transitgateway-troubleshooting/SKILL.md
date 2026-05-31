---
name: transitgateway-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Transit Gateway (TGW)
  problems by analyzing TGW creation, route table configuration, attachment
  failures, VPC attachment issues, subnet routing, cross-AZ connectivity,
  inter-region peering, peering route propagation, VPN attachments, BGP over
  TGW, Direct Connect gateway attachments, transit VIF configuration, route
  propagation, blackhole routes, route table associations, multicast domains,
  group membership, and following structured runbooks. Activate when: TGW
  creation failures, route table issues, attachment failures, VPC attachment
  problems, subnet routing errors, cross-AZ connectivity issues, inter-region
  peering failures, peering route propagation not working, VPN attachment
  errors, BGP session failures over TGW, DX gateway attachment issues,
  transit VIF problems, route propagation failures, blackhole routes blocking
  traffic, route table association errors, multicast domain issues, multicast
  group membership problems, or the user says something is wrong with Transit
  Gateway without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with EC2, RAM, CloudWatch, CloudTrail,
  Direct Connect, and optionally VPC Flow Logs permissions. Some operations
  require TGW owner or shared account permissions.
---

# AWS Transit Gateway Diagnostics

## When to use

Any AWS Transit Gateway investigation where the console alone is insufficient — TGW creation, route table configuration, VPC/VPN/DX attachments, route propagation, blackhole routes, inter-region peering, multicast, or performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id>
aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=<tgw-id>
aws ec2 describe-transit-gateway-route-tables --filters Name=transit-gateway-id,Values=<tgw-id>
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb-id> --filters Name=state,Values=active,blackhole
```

### Step 2 — Domain deep dive

```
aws ec2 describe-transit-gateway-vpc-attachments --filters Name=transit-gateway-id,Values=<tgw-id>
aws ec2 describe-transit-gateway-peering-attachments --filters Name=transit-gateway-id,Values=<tgw-id>
aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <rtb-id>
aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <rtb-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
aws ec2 describe-transit-gateway-multicast-domains --filters Name=transit-gateway-id,Values=<tgw-id>
aws cloudwatch get-metric-statistics --namespace AWS/TransitGateway --metric-name BytesIn --dimensions Name=TransitGateway,Value=<tgw-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/transitgateway-guardrails.md` before concluding on any Transit Gateway issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-transit-gateways` | Check TGW status, configuration, ASN |
| `describe-transit-gateway-attachments` | List all attachments and their states |
| `describe-transit-gateway-vpc-attachments` | Check VPC attachment details and subnets |
| `describe-transit-gateway-route-tables` | List route tables and their configuration |
| `search-transit-gateway-routes` | Search routes including blackhole detection |
| `get-transit-gateway-route-table-associations` | Check route table associations |
| `get-transit-gateway-route-table-propagations` | Check route propagation settings |
| `describe-transit-gateway-peering-attachments` | Check peering attachment status |
| `describe-transit-gateway-multicast-domains` | Check multicast domain configuration |
| `describe-vpn-connections` | Check VPN attachment status and tunnels |
| `describe-direct-connect-gateway-attachments` | Check DX gateway attachment |
| CloudWatch Metrics | Monitor bandwidth, packets, bytes |

## Gotchas: AWS Transit Gateway

- TGW route tables are completely separate from VPC route tables. A VPC route table entry pointing to a TGW only gets traffic TO the TGW. The TGW route table then determines where traffic goes NEXT. Both must be configured correctly for end-to-end connectivity.
- Attachments need a subnet in each AZ where you want connectivity. If a VPC attachment only has a subnet in AZ-a, instances in AZ-b of that VPC cannot reach the TGW. Always specify subnets in all AZs where instances reside.
- Route propagation vs static routes: propagated routes are automatically learned from attachments. Static routes are manually configured and take precedence over propagated routes for the same CIDR. Mixing both can cause confusion.
- Blackhole routes block traffic silently. They appear when an attachment is deleted but the route remains, or when manually created. Always check for blackhole routes when debugging connectivity failures.
- Inter-region peering has bandwidth limits and does NOT support route propagation. You must create static routes for peered TGWs. Peering also does not support multicast.
- Each attachment can only associate with ONE route table. Association determines which route table is used for routing decisions for traffic FROM that attachment. A route table can have many attachments associated.
- Appliance mode must be enabled on VPC attachments for stateful inspection appliances. Without it, return traffic may use a different AZ path, breaking stateful firewalls.
- MTU 8500 is supported within the same region for VPC attachments. VPN attachments support 1500 MTU. Inter-region peering supports 8500 MTU. Mismatched MTU causes packet drops.

## Anti-hallucination rules

1. Always cite specific TGW IDs, attachment IDs, route table IDs, or API responses as evidence.
2. TGW route tables are NOT VPC route tables. Never confuse the two.
3. Inter-region peering does NOT support route propagation. Never suggest enabling propagation on peering attachments.
4. Each attachment associates with exactly ONE route table. Never suggest multiple associations.
5. Multicast support must be enabled at TGW creation. Never suggest enabling it on an existing TGW.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — TGW | A1–A3 | Creation failures, route table issues, attachment failures |
| B — VPC | B1–B3 | VPC attachment, subnet routing, cross-AZ |
| C — Peering | C1–C2 | Inter-region peering, peering route propagation |
| D — VPN | D1–D2 | VPN attachment, BGP over TGW |
| E — Direct Connect | E1–E2 | DX gateway attachment, transit VIF |
| F — Routing | F1–F3 | Route propagation, blackhole routes, route table association |
| G — Multicast | G1–G2 | Multicast domain, group membership |
| Z — Catch-All | Z1 | General troubleshooting |
