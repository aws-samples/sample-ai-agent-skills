---
title: "B3 â€” VPN Routing Issues"
description: "Diagnose routing problems over Site-to-Site VPN connections"
status: active
severity: HIGH
triggers:
  - "VPN routes missing"
  - "can't reach VPC over VPN"
  - "VPN routing"
  - "static routes VPN"
  - "BGP routes VPN"
owner: devops-agent
objective: "Identify VPN routing misconfigurations and restore end-to-end connectivity"
context: "VPN connections support static routing or dynamic routing (BGP). For static routing, routes must be manually configured on both sides. For BGP, routes are exchanged automatically but require proper BGP configuration. VPC route tables must have route propagation enabled or static routes pointing to the VGW/TGW."
---

## Phase 1 â€” Triage

MUST:
- Check VPN routing type: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].Options.StaticRoutesOnly'`
- Check VPN routes: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].Routes'`
- Check VPC route tables for VPN routes: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id> --query 'RouteTables[].Routes[?GatewayId!=`null`]'`
- Verify route propagation is enabled: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id> --query 'RouteTables[].PropagatingVgws'`

SHOULD:
- Check if the customer gateway is advertising the correct prefixes (for BGP)
- Verify no overlapping CIDR blocks between VPC and on-premises
- Check for more specific static routes overriding propagated routes

MAY:
- Check customer gateway routing table for return routes to VPC CIDR
- Verify security groups and NACLs allow the traffic

## Phase 2 â€” Remediate

MUST:
- Enable route propagation on VPC route tables that need VPN routes
- For static routing: add static routes on the VPN connection for on-premises CIDRs
- Ensure customer gateway has return routes to the VPC CIDR

SHOULD:
- Use BGP dynamic routing for automatic route exchange
- Verify no route table conflicts (static routes take precedence over propagated)
- Check that the most specific route is pointing to the correct target

MAY:
- Add static routes as backup if BGP routes are not propagating
- Review route table associations to ensure correct subnets use the right route table

## Common Issues

- symptoms: "VPN tunnel is UP but cannot reach VPC resources"
  diagnosis: "Route propagation not enabled on VPC route table, or missing static routes."
  resolution: "Enable route propagation on the VPC route table or add static routes to VGW."

- symptoms: "Can reach VPC from on-premises but not the reverse"
  diagnosis: "Customer gateway missing return route to VPC CIDR."
  resolution: "Add route to VPC CIDR on the customer gateway pointing through the VPN tunnel."

- symptoms: "BGP routes not appearing in VPC route table"
  diagnosis: "BGP session not established, or route propagation not enabled."
  resolution: "Verify BGP is UP (see A3 runbook) and enable route propagation on the route table."

## Output Format

```yaml
root_cause: "vpn_routing_issue â€” <specific_cause>"
evidence:
  - type: vpn_routes
    content: "<VPN connection routes>"
  - type: route_tables
    content: "<VPC route table entries>"
severity: HIGH
mitigation:
  immediate: "Fix route propagation or add missing routes"
  long_term: "Use BGP dynamic routing and monitor route table changes"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — Direct Connect and VPN provide critical network connectivity |
| Data Risk | HIGH — Network path changes can disrupt all traffic flows |
| Recovery Complexity | HIGH — Physical cross-connects and BGP sessions require coordination |
| Blast Radius | CRITICAL — Affects all workloads dependent on hybrid connectivity |

## Escalation Conditions

- Connection down with no redundant path available — escalate immediately to network operations
- BGP session flapping causing route instability across multiple VIFs
- Physical layer issues requiring colocation provider coordination
- MACsec or IPsec negotiation failures blocking encrypted traffic
- Transit Gateway attachment issues affecting multiple VPCs
- Failover not triggering between DX and VPN backup paths

## Data Sensitivity

- BGP route advertisements may expose internal network topology
- VPN pre-shared keys and IKE credentials must never be logged or displayed
- Connection and VIF IDs should be treated as sensitive infrastructure identifiers
- MACsec secret keys (CKN/CAK) must be handled with encryption-at-rest standards
- Customer router configurations may contain proprietary network architecture details

## PROHIBITED Actions

- NEVER suggest deleting a Direct Connect connection without confirming backup connectivity
- NEVER modify BGP route advertisements without understanding the full routing topology
- NEVER change VPN tunnel configurations on both tunnels simultaneously
- NEVER disable MACsec encryption on a connection that requires encrypted transit
- NEVER delete a Virtual Interface (VIF) that is actively carrying production traffic
- NEVER modify Transit Gateway route tables without verifying all dependent attachments

## Phase 3 — Rollback

- If BGP changes cause route instability: revert route advertisements to previous AS-path and prefix configuration
- If VPN tunnel changes break connectivity: restore previous IKE/IPsec parameters from backup configuration
- If VIF deletion was premature: recreate VIF with identical VLAN, BGP ASN, and auth key settings
- If failover configuration changes fail: restore previous DX/VPN failover priority and health check settings
- If Transit Gateway route changes cause blackholing: revert route table associations and propagations
- Document all rollback actions in the incident timeline with before/after routing state
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest disabling MACsec encryption"
  - "NEVER suggest removing BGP authentication"
  - "NEVER modify production VPN tunnels without maintenance window"
