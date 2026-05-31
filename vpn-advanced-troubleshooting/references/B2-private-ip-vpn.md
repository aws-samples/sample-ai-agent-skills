---
title: "B2 — Private IP VPN"
description: "Diagnose private IP VPN connection issues over AWS Direct Connect"
status: active
severity: HIGH
triggers: ["private IP VPN failed", "private VPN not connecting", "VPN over Direct Connect error", "private IP VPN tunnel down"]
owner: devops-agent
objective: "Resolve private IP VPN connection issues running over AWS Direct Connect"
context: "Private IP VPN uses private IP addresses and runs over Direct Connect. Requires a transit gateway with a Direct Connect gateway association. Does NOT work over the public internet."
---

## Phase 1 — Triage

MUST:
- Check VPN connection details: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].{Id:VpnConnectionId,State:State,TgwId:TransitGatewayId,GatewayAssociation:GatewayAssociationState}'`
- Verify Direct Connect connection is AVAILABLE: `aws directconnect describe-connections --query 'connections[*].{Id:connectionId,State:connectionState,Bandwidth:bandwidth}'`
- Check DX gateway association with TGW: `aws directconnect describe-direct-connect-gateway-associations --direct-connect-gateway-id <dx-gw-id>`

SHOULD:
- Verify the DX virtual interface is UP
- Check tunnel telemetry: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`
- Verify BGP sessions on the DX virtual interface

MAY:
- Check CloudWatch metrics for DX connection health
- Review the DX gateway route table for VPN prefixes

## Phase 2 — Remediate

MUST:
- Ensure Direct Connect connection is AVAILABLE and virtual interface is UP
- Verify DX gateway is associated with the transit gateway
- Confirm the VPN connection is configured for private IP (not public)

SHOULD:
- Verify BGP sessions on both the DX virtual interface and VPN tunnels
- Check for IP address conflicts between private VPN tunnel IPs and DX IPs

MAY:
- Set up redundant DX connections for private IP VPN high availability
- Configure BFD (Bidirectional Forwarding Detection) on the DX connection

## Common Issues

- symptoms: "Private IP VPN tunnels DOWN despite DX connection being AVAILABLE"
  diagnosis: "DX gateway not associated with the transit gateway or virtual interface DOWN."
  resolution: "Associate the DX gateway with the TGW and ensure the transit virtual interface is UP."

- symptoms: "Cannot create private IP VPN connection"
  diagnosis: "Missing DX gateway association with the transit gateway."
  resolution: "Create a DX gateway, associate it with the TGW, then create the private IP VPN connection."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN details"
  - "describe-connections: GREEN — read-only DX details"
  - "describe-direct-connect-gateway-associations: GREEN — read-only association details"
  - "create-vpn-connection (private IP): YELLOW — creates VPN resource over DX"
  - "delete-vpn-connection: RED — removes VPN connectivity"
```

## Escalation Conditions

- Direct Connect connection is DOWN affecting all private IP VPN tunnels
- DX gateway association fails or is in an unexpected state
- Private IP VPN requires DX infrastructure changes managed by another team
- IP address conflicts between DX and VPN tunnel addressing

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN and tunnel configuration
- `aws directconnect describe-connections`: HIGH — reveals DX connection details and location
- `aws directconnect describe-direct-connect-gateway-associations`: MEDIUM — reveals DX-TGW topology

## Prohibited Actions

- NEVER attempt to create a private IP VPN without an active Direct Connect connection
- NEVER modify DX gateway associations without understanding the impact on all connected VPNs
- NEVER assume private IP VPN works over the public internet
- NEVER delete DX virtual interfaces that carry private IP VPN traffic

## Phase 3 — Rollback

1. If a private IP VPN was created in error, delete it: `aws ec2 delete-vpn-connection --vpn-connection-id <vpn-id>`
2. If DX gateway association was modified, restore the previous association
3. Verify the Direct Connect connection and virtual interfaces are unaffected
4. Confirm other VPN connections over the same DX are still functional
5. Document the rollback and notify the network and DX teams

## Output Format

```yaml
root_cause: "private_ip_vpn — <specific_cause>"
evidence:
  - type: vpn_dx_state
    content: "<VPN and Direct Connect details>"
severity: HIGH
mitigation:
  immediate: "Fix private IP VPN or DX configuration"
  long_term: "Implement redundant DX for private IP VPN high availability"
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
