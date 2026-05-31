---
title: "F1 — Multi-Tunnel Load Balancing"
description: "Diagnose multi-tunnel and multi-connection VPN load balancing issues"
status: active
severity: HIGH
triggers: ["VPN load balancing not working", "traffic on one tunnel only", "uneven VPN distribution", "multi-tunnel issue"]
owner: devops-agent
objective: "Resolve multi-tunnel and multi-connection VPN traffic distribution issues"
context: "Each VPN connection has 2 tunnels. For load balancing across tunnels, ECMP with multiple VPN connections on a TGW is required. Single-connection tunnels are active/passive (VGW) or active/active (TGW with ECMP)."
---

## Phase 1 — Triage

MUST:
- Check VPN connection type (VGW vs TGW): `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].{Id:VpnConnectionId,TgwId:TransitGatewayId,VgwId:VpnGatewayId}'`
- Check tunnel status for all connections: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`
- Check per-tunnel throughput: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelDataIn --dimensions Name=VpnId,Value=<vpn-id> Name=TunnelIpAddress,Value=<tunnel-ip> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- For TGW: verify ECMP is enabled: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.VpnEcmpSupport'`
- Count the number of VPN connections to the TGW
- Verify BGP is used (required for ECMP)

MAY:
- Analyze traffic flow patterns to understand per-flow distribution
- Check customer gateway load balancing configuration

## Phase 2 — Remediate

MUST:
- For VGW VPN: understand that only one tunnel is active at a time (active/passive)
- For TGW VPN with ECMP: ensure multiple VPN connections exist and ECMP is enabled
- Verify all tunnels are UP across all VPN connections

SHOULD:
- Remove AS path prepending that causes unequal route preference
- Ensure the same prefixes are advertised from all customer gateways

MAY:
- Add additional VPN connections for higher aggregate throughput
- Implement traffic engineering on the customer gateway for optimal distribution

## Common Issues

- symptoms: "Only one tunnel carries traffic on a VGW-based VPN"
  diagnosis: "VGW VPN uses active/passive tunnels. This is expected behavior."
  resolution: "This is by design for VGW VPNs. For active/active, migrate to TGW with ECMP."

- symptoms: "TGW VPN with ECMP but traffic still on one connection"
  diagnosis: "Single large flow uses one path (ECMP is per-flow). Or BGP routes have different AS path lengths."
  resolution: "ECMP distributes per-flow. Verify AS path lengths are equal. Multiple concurrent flows will distribute."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN details"
  - "describe-transit-gateways: GREEN — read-only TGW details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "create-vpn-connection: YELLOW — creates additional VPN resource"
  - "modify-transit-gateway (ECMP): YELLOW — changes TGW routing behavior"
```

## Escalation Conditions

- Single tunnel approaching 1.25 Gbps throughput limit with no load balancing
- ECMP not distributing traffic despite correct configuration
- Migration from VGW to TGW required for active/active tunnels
- Customer gateway does not support multiple VPN connections for ECMP

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration
- `aws ec2 describe-transit-gateways`: MEDIUM — reveals TGW configuration
- `aws cloudwatch get-metric-statistics`: LOW — reveals throughput metrics

## Prohibited Actions

- NEVER assume VGW VPN supports active/active tunnels — it is always active/passive
- NEVER remove VPN connections from an ECMP group without verifying remaining capacity
- NEVER claim per-packet load balancing — ECMP is per-flow
- NEVER modify TGW ECMP settings without understanding the impact on all VPN connections

## Phase 3 — Rollback

1. If additional VPN connections were created, delete them if not needed
2. If TGW ECMP was modified, revert the setting
3. Verify traffic is flowing through the remaining VPN connections
4. Confirm throughput is sufficient for the workload
5. Document the load balancing configuration and capacity plan

## Output Format

```yaml
root_cause: "multi_tunnel_lb — <specific_cause>"
evidence:
  - type: tunnel_distribution
    content: "<per-tunnel throughput and ECMP details>"
severity: HIGH
mitigation:
  immediate: "Fix load balancing configuration"
  long_term: "Implement ECMP with capacity monitoring"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
