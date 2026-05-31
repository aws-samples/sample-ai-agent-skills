---
name: vpn-advanced-troubleshooting
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot advanced AWS
  Site-to-Site VPN problems including accelerated VPN, ECMP,
  certificate-based authentication, private IP VPN, CloudWatch
  monitoring, log analysis, BGP route propagation, static route
  failover, tunnel health checks, DPD timeouts, and multi-tunnel
  load balancing using structured runbooks.
  Activate when: accelerated VPN issues, ECMP configuration problems,
  certificate-based auth failures, private IP VPN errors, VPN
  monitoring gaps, log analysis needs, BGP propagation issues,
  static route failover failures, tunnel health check problems,
  DPD timeout issues, multi-tunnel load balancing errors, or the
  user says something is wrong with Site-to-Site VPN.
compatibility: >
  Requires AWS CLI or SDK access with ec2, acm-pca, cloudwatch,
  logs, cloudtrail, and directconnect permissions.
---

# AWS Site-to-Site VPN Advanced Diagnostics

## When to use

Any advanced AWS Site-to-Site VPN investigation — accelerated VPN, ECMP configuration, certificate-based authentication, private IP VPN, CloudWatch monitoring, VPN log analysis, BGP route propagation, static route failover, tunnel health checks, DPD timeouts, multi-tunnel load balancing, or general VPN troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-vpn-connections --query 'VpnConnections[*].{Id:VpnConnectionId,State:State,Type:Type,Category:Category,GatewayAssociation:TransitGatewayId}'
aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'
aws ec2 describe-vpn-gateways --query 'VpnGateways[*].{Id:VpnGatewayId,State:State,AZ:AvailabilityZone}'
```

### Step 2 — Domain deep dive

```
aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Options'
aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --start-time <epoch-ms> --limit 50
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
aws ec2 describe-customer-gateways --customer-gateway-ids <cgw-id>
aws ec2 describe-transit-gateway-route-tables --transit-gateway-route-table-ids <tgw-rtb-id>
```

Read `references/guardrails.md` before concluding on any VPN issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ec2 describe-vpn-connections` | Get VPN connection details and tunnel telemetry |
| `ec2 describe-customer-gateways` | Get customer gateway configuration |
| `ec2 describe-vpn-gateways` | Get virtual private gateway details |
| `cloudwatch get-metric-statistics` | Get VPN tunnel metrics |
| `logs filter-log-events` | Analyze VPN connection logs |
| `ec2 describe-transit-gateway-route-tables` | Check TGW route propagation |
| `ec2 search-transit-gateway-routes` | Search for specific routes in TGW |

## Gotchas: AWS Site-to-Site VPN Advanced

- Accelerated VPN uses AWS Global Accelerator for improved performance. It requires a transit gateway (not VGW) and is not available in all regions.
- ECMP (Equal-Cost Multi-Path) is only supported with transit gateway VPN attachments. VGW-based VPNs do not support ECMP. ECMP requires multiple VPN connections.
- Certificate-based authentication uses ACM Private CA certificates instead of pre-shared keys. The certificate must be issued by an ACM PCA that is in the same region.
- Private IP VPN runs over Direct Connect using private IP addresses. It requires a transit gateway and a Direct Connect gateway association.
- VPN tunnel logs (when enabled) are sent to CloudWatch Logs. Log delivery has a delay of up to 5 minutes. Not all tunnel events are logged.
- BGP route propagation from on-premises to AWS can take up to 60 seconds. Route flapping triggers BGP dampening which can delay convergence further.
- DPD (Dead Peer Detection) timeout defaults vary by customer gateway vendor. Mismatched DPD settings between AWS and the customer gateway cause tunnel flapping.

## Anti-hallucination rules

1. Always cite specific VPN connection IDs, tunnel endpoint IPs, or API responses as evidence.
2. Accelerated VPN requires a transit gateway. Never suggest accelerated VPN with a virtual private gateway.
3. ECMP requires multiple VPN connections to a transit gateway. Never claim a single VPN connection supports ECMP.
4. VPN tunnel telemetry shows UP/DOWN status. Never conflate tunnel state with data plane throughput.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Accelerated & ECMP | A1–A2 | Accelerated VPN issues, ECMP configuration |
| B — Authentication | B1–B2 | Certificate-based auth, private IP VPN |
| C — Monitoring | C1–C2 | CloudWatch monitoring, VPN log analysis |
| D — Routing | D1–D2 | BGP route propagation, static route failover |
| E — Tunnel Health | E1–E2 | Tunnel health checks, DPD timeout issues |
| F — Load Balancing | F1 | Multi-tunnel load balancing |
| Z — Catch-All | Z1 | General troubleshooting |
