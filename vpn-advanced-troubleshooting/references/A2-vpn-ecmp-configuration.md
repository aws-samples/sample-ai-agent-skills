---
title: "A2 — VPN ECMP Configuration"
description: "Diagnose ECMP (Equal-Cost Multi-Path) configuration issues for VPN connections"
status: active
severity: HIGH
triggers: ["ECMP not working", "VPN load balancing failed", "traffic not distributed", "ECMP configuration error"]
owner: devops-agent
objective: "Resolve ECMP configuration and traffic distribution issues for VPN connections"
context: "ECMP distributes traffic across multiple VPN connections attached to a transit gateway. Requires multiple VPN connections (not just multiple tunnels) and BGP dynamic routing."
---

## Phase 1 — Triage

MUST:
- Verify TGW has ECMP enabled: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.VpnEcmpSupport'`
- List VPN connections on the TGW: `aws ec2 describe-vpn-connections --filters Name=transit-gateway-id,Values=<tgw-id> --query 'VpnConnections[*].{Id:VpnConnectionId,State:State,Routing:Options.StaticRoutesOnly}'`
- Verify multiple VPN connections exist (ECMP requires more than one)
- Check all tunnels are UP: `aws ec2 describe-vpn-connections --filters Name=transit-gateway-id,Values=<tgw-id> --query 'VpnConnections[*].VgwTelemetry'`

SHOULD:
- Verify BGP is used (StaticRoutesOnly=false) — ECMP requires BGP
- Check BGP routes on the TGW route table: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb-id> --filters Name=type,Values=propagated`
- Verify the same prefixes are advertised from all VPN connections

MAY:
- Check CloudWatch metrics per VPN connection to verify traffic distribution
- Review customer gateway BGP configuration for AS path prepending that may affect ECMP

## Phase 2 — Remediate

MUST:
- Enable ECMP on the TGW if disabled (requires TGW modification or recreation)
- Ensure all VPN connections use BGP (not static routing)
- Verify the same BGP prefixes are advertised from all customer gateways

SHOULD:
- Remove AS path prepending that may cause unequal traffic distribution
- Ensure all VPN connections have both tunnels UP for maximum throughput

MAY:
- Add additional VPN connections for higher aggregate throughput
- Implement per-flow monitoring to verify ECMP distribution

## Common Issues

- symptoms: "All traffic goes through one VPN connection despite ECMP enabled"
  diagnosis: "BGP routes have different AS path lengths due to AS path prepending on some connections."
  resolution: "Remove AS path prepending on all VPN connections to ensure equal-cost routes."

- symptoms: "ECMP not distributing traffic evenly"
  diagnosis: "ECMP uses per-flow hashing. A single large flow will use one path."
  resolution: "ECMP distributes per-flow, not per-packet. Multiple concurrent flows will be distributed across connections."

## Safety Ratings

```
safety_ratings:
  - "describe-transit-gateways: GREEN — read-only TGW details"
  - "describe-vpn-connections: GREEN — read-only VPN details"
  - "search-transit-gateway-routes: GREEN — read-only route search"
  - "modify-transit-gateway (ECMP): YELLOW — changes TGW routing behavior"
  - "create-vpn-connection: YELLOW — creates additional VPN resource"
```

## Escalation Conditions

- ECMP cannot be enabled on the TGW without recreation (affecting all attachments)
- Traffic distribution is severely unbalanced despite correct ECMP configuration
- Customer gateway vendor does not support the required BGP configuration for ECMP
- Adding VPN connections exceeds the TGW VPN attachment quota

## Data Sensitivity

- `aws ec2 describe-transit-gateways`: MEDIUM — reveals TGW configuration
- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration details
- `aws ec2 search-transit-gateway-routes`: MEDIUM — reveals routing topology

## Prohibited Actions

- NEVER modify TGW ECMP settings in production without understanding the impact on all VPN connections
- NEVER assume ECMP provides per-packet load balancing — it is per-flow
- NEVER create VPN connections with static routing for ECMP — BGP is required
- NEVER remove VPN connections from an ECMP group without verifying remaining capacity

## Phase 3 — Rollback

1. If TGW ECMP was modified, revert the setting: `aws ec2 modify-transit-gateway --transit-gateway-id <tgw-id> --options VpnEcmpSupport=disable`
2. If additional VPN connections were created, delete them if not needed
3. Verify traffic is flowing through the remaining VPN connections
4. Restore customer gateway BGP configuration to pre-change state
5. Document the rollback and verify throughput is sufficient

## Output Format

```yaml
root_cause: "vpn_ecmp — <specific_cause>"
evidence:
  - type: ecmp_config
    content: "<TGW ECMP and VPN connection details>"
severity: HIGH
mitigation:
  immediate: "Fix ECMP configuration"
  long_term: "Implement ECMP monitoring and capacity planning"
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
