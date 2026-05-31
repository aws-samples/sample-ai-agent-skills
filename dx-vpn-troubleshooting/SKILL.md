---
name: dx-vpn-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Direct Connect
  and Site-to-Site VPN problems by analyzing connections, virtual
  interfaces, VPN tunnels, BGP sessions, and following structured
  runbooks. Activate when: DX connection down, VIF creation failures,
  BGP session issues, LAG problems, VPN tunnel down, IKE negotiation
  failures, routing issues, throughput limits, failover problems,
  Transit Gateway attachment issues, MACsec encryption, latency,
  or the user says something is wrong with Direct Connect or VPN
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with DirectConnect, EC2, CloudWatch,
  and optionally Transit Gateway permissions.
---

# Direct Connect & Site-to-Site VPN Diagnostics

## When to use

Any Direct Connect or Site-to-Site VPN investigation where the console alone is insufficient — connection failures, VIF provisioning, BGP session debugging, VPN tunnel establishment, IKE/IPsec negotiation, routing propagation, failover validation, Transit Gateway integration, encryption, or performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws directconnect describe-connections
aws directconnect describe-virtual-interfaces
aws ec2 describe-vpn-connections
aws ec2 describe-vpn-gateways
aws ec2 describe-customer-gateways
aws directconnect describe-lags
aws directconnect describe-direct-connect-gateways
```

### Step 2 — Domain deep dive

```
aws ec2 describe-transit-gateways
aws ec2 describe-transit-gateway-attachments
aws ec2 describe-transit-gateway-route-tables
aws ec2 describe-route-tables --filters Name=route.origin,Values=EnableVgwRoutePropagation
aws cloudwatch get-metric-statistics --namespace AWS/DX --metric-name ConnectionState --dimensions Name=ConnectionId,Value=<dxcon-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/DX --metric-name ConnectionBpsIngress --dimensions Name=ConnectionId,Value=<dxcon-id> ...
```

Read `references/dx-vpn-guardrails.md` before concluding on any Direct Connect or VPN issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-connections` | DX connection state, bandwidth, location |
| `describe-virtual-interfaces` | VIF type, VLAN, BGP peer status |
| `describe-vpn-connections` | VPN tunnel status, IKE config, routing |
| `describe-vpn-gateways` | VGW attachment, availability zone |
| `describe-customer-gateways` | CGW IP, BGP ASN |
| `describe-lags` | LAG member connections, minimum links |
| `describe-direct-connect-gateways` | DX gateway associations |
| `describe-transit-gateways` | TGW config, route tables |
| CloudWatch DX/VPN metrics | Connection state, tunnel state, throughput |

## Gotchas: Direct Connect & Site-to-Site VPN

- DX is NOT encrypted by default. Traffic traverses a dedicated line but is unencrypted. Use MACsec (802.1AE) for Layer 2 encryption on supported connections (10 Gbps and 100 Gbps) or run a VPN over DX for IPsec encryption.
- BGP session establishment requires matching ASN, BGP auth key, and correct VLAN tagging. AWS provides the BGP peer IP addresses — the customer router must be configured to peer with the AWS-side IP.
- VIF types matter: Private VIF connects to a single VPC via VGW, Public VIF accesses AWS public services (S3, DynamoDB endpoints), Transit VIF connects to Transit Gateway for multi-VPC access.
- DX failover to VPN requires proper BGP configuration. The VPN route must have a longer AS path or lower local preference so DX is preferred. Without correct BGP attributes, failover may not work as expected.
- VPN tunnel has a 1.25 Gbps limit per tunnel. This is a hard AWS limit. For higher throughput, use multiple VPN connections with ECMP on Transit Gateway.
- VPN uses two tunnels for redundancy — both should be UP. AWS performs maintenance on one tunnel at a time. If only one tunnel is configured, connectivity is lost during maintenance.
- LAG (Link Aggregation Group) requires all member connections to have the same bandwidth and terminate at the same AWS Direct Connect location. Mixing speeds or locations is not supported.
- DX Gateway enables multi-VPC and multi-region connectivity over a single DX connection. Without a DX gateway, each VPC requires its own VIF.
- Transit VIF vs Private VIF for Transit Gateway: Transit VIF connects DX to TGW for scalable multi-VPC routing. Private VIF connects to a single VPC via VGW. Use Transit VIF when connecting to multiple VPCs.

### VIF type comparison

| VIF Type | Connects To | Use Case |
|----------|-------------|----------|
| Private VIF | VPC (via VGW or DX Gateway) | Single VPC private connectivity |
| Public VIF | AWS public services | Access S3, DynamoDB, public endpoints |
| Transit VIF | Transit Gateway (via DX Gateway) | Multi-VPC, multi-region connectivity |

### VPN tunnel limits

| Parameter | Limit |
|-----------|-------|
| Throughput per tunnel | 1.25 Gbps |
| Packets per second per tunnel | 140,000 |
| Maximum tunnels per VPN connection | 2 |
| VPN connections per VGW | 10 |
| VPN connections per TGW | Up to thousands (with ECMP) |

## Anti-hallucination rules

1. Always cite specific connection states, BGP peer status, tunnel status, or CloudWatch metrics as evidence.
2. DX is NOT encrypted by default. Never claim DX provides encryption without MACsec or VPN overlay.
3. VPN has a 1.25 Gbps per-tunnel hard limit. Never suggest a single VPN tunnel can exceed this.
4. Both VPN tunnels should be UP. Never dismiss a single-tunnel-down state as acceptable for production.
5. LAG members must have the same speed and location. Never suggest mixing connection speeds in a LAG.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 28 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Direct Connect | A1-A4 | Connection down, VIF creation failures, BGP session issues, LAG problems |
| B — VPN | B1-B4 | Tunnel down, IKE negotiation failures, routing issues, throughput limits |
| C — Routing | C1-C3 | BGP route propagation, route table issues, asymmetric routing |
| D — Failover | D1-D2 | DX-to-VPN failover, active-active vs active-passive |
| E — Transit Gateway | E1-E2 | TGW attachment issues, TGW route table |
| F — Security | F1-F2 | MACsec encryption, VPN encryption |
| G — Performance | G1-G2 | Bandwidth issues, latency |
| H — Migration | H1-H2 | DX setup/provisioning, VPN migration |
| Z — Catch-All | Z1 | General troubleshooting |
