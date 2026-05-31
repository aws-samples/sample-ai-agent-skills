---
title: "C1 â€” BGP Route Propagation"
description: "Diagnose BGP route propagation failures between on-premises and AWS"
status: active
severity: HIGH
triggers:
  - "routes not propagating"
  - "BGP routes missing"
  - "no routes from on-premises"
  - "VPC routes not received"
owner: devops-agent
objective: "Identify why BGP routes are not propagating and restore route exchange"
context: "BGP route propagation issues can occur at multiple levels: BGP session not established, routes not advertised by the customer, routes not accepted by AWS (prefix limits, invalid prefixes), or VPC route table propagation not enabled. AWS advertises VPC CIDR blocks over BGP. Customers must advertise their on-premises prefixes."
---

## Phase 1 â€” Triage

MUST:
- Verify BGP session is UP: `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id> --query 'virtualInterfaces[].bgpPeers[].bgpStatus'`
- Check VPC route tables for propagated routes: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`
- Verify route propagation is enabled: check `PropagatingVgws` in route table output
- For VPN: check routes in VPN connection: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].Routes'`

SHOULD:
- Check the number of routes advertised (AWS has a 100-prefix limit per BGP session by default)
- Verify customer is advertising valid, non-overlapping prefixes
- Check if route filtering is applied on either side

MAY:
- Check customer router BGP table for received and advertised routes
- Verify no AS path filtering is blocking routes

## Phase 2 â€” Remediate

MUST:
- Ensure BGP session is established (see A3 runbook if not)
- Enable route propagation on all VPC route tables that need on-premises routes
- Verify customer gateway is advertising the correct prefixes

SHOULD:
- Summarize on-premises routes to stay within the 100-prefix limit
- Remove any route filters that may be blocking valid prefixes
- Check for route conflicts between static and propagated routes

MAY:
- Request a prefix limit increase via AWS Support if needed
- Implement route summarization on the customer router

## Common Issues

- symptoms: "BGP is UP but no on-premises routes in VPC route table"
  diagnosis: "Route propagation not enabled on the VPC route table."
  resolution: "Enable VGW route propagation on the route table: modify route table propagation settings."

- symptoms: "Only some on-premises routes appear in VPC"
  diagnosis: "Exceeding the 100-prefix limit per BGP session, or route filtering."
  resolution: "Summarize routes on the customer side. Check for prefix filters."

- symptoms: "On-premises cannot reach VPC despite BGP being UP"
  diagnosis: "AWS is advertising VPC CIDR but customer router is not installing the route."
  resolution: "Check customer router BGP table for received routes. Verify no inbound route filters."

## Output Format

```yaml
root_cause: "bgp_route_propagation â€” <specific_cause>"
evidence:
  - type: bgp_status
    content: "<BGP peer state>"
  - type: route_tables
    content: "<VPC route table entries and propagation settings>"
severity: HIGH
mitigation:
  immediate: "Enable route propagation or fix BGP advertisement"
  long_term: "Implement route summarization and monitoring"
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
