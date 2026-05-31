---
title: "A3 â€” BGP Session Issues"
description: "Diagnose BGP session establishment and stability problems on Direct Connect"
status: active
severity: HIGH
triggers:
  - "BGP down"
  - "BGP not established"
  - "BGP flapping"
  - "peer not reachable"
  - "routes not advertised"
owner: devops-agent
objective: "Identify why the BGP session is not establishing or is unstable and restore BGP peering"
context: "BGP session issues on DX are caused by mismatched ASN, incorrect peer IPs, wrong VLAN tagging, BGP auth key mismatch, or customer router misconfiguration. AWS provides the BGP peer configuration â€” the customer must match it exactly."
---

## Phase 1 â€” Triage

MUST:
- Check VIF BGP peer status: `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id> --query 'virtualInterfaces[].bgpPeers'`
- Verify VIF state is "available": `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id> --query 'virtualInterfaces[].virtualInterfaceState'`
- Check BGP peer state (should be "up"): look at `bgpPeerState` and `bgpStatus` in the output
- Verify customer-side BGP configuration matches AWS-provided values (ASN, peer IP, auth key)

SHOULD:
- Check CloudWatch BGP metrics for the VIF
- Verify VLAN tagging on the customer router matches the VIF VLAN ID
- Check for BGP hold timer expiration on the customer router

MAY:
- Check customer router BGP logs for specific error messages
- Verify MTU settings (DX supports jumbo frames 9001 bytes but both sides must agree)

## Phase 2 â€” Remediate

MUST:
- Match the customer router BGP ASN to the value configured on the VIF
- Configure the correct AWS-side and customer-side BGP peer IP addresses
- Set the correct BGP authentication key if configured
- Ensure VLAN tagging matches the VIF VLAN ID

SHOULD:
- Set BGP hold timer to 90 seconds (AWS default) or match AWS-side timer
- Enable BFD (Bidirectional Forwarding Detection) for faster failover
- Verify the customer router is advertising the correct prefixes

MAY:
- Reset the BGP session from the customer router
- Enable BGP debug logging temporarily for troubleshooting

## Common Issues

- symptoms: "BGP peer state is 'down', bgpStatus is 'down'"
  diagnosis: "BGP session not establishing. Likely ASN mismatch, incorrect peer IPs, or VLAN tagging issue."
  resolution: "Verify customer router config matches AWS-provided BGP parameters exactly."

- symptoms: "BGP session establishes then drops repeatedly (flapping)"
  diagnosis: "Hold timer expiration, MTU mismatch causing packet drops, or unstable physical layer."
  resolution: "Check hold timer settings, verify MTU consistency, check physical layer stability."

- symptoms: "BGP is up but no routes received from AWS"
  diagnosis: "VGW not attached to VPC, or VPC CIDR not being advertised."
  resolution: "Verify VGW is attached to the VPC. Check that the VPC has a CIDR block configured."

## Output Format

```yaml
root_cause: "bgp_session_issue â€” <specific_cause>"
evidence:
  - type: bgp_peer_status
    content: "<BGP peer state and configuration>"
  - type: vif_config
    content: "<VIF VLAN, ASN, peer IPs>"
severity: HIGH
mitigation:
  immediate: "Fix BGP configuration mismatch and re-establish session"
  long_term: "Document BGP configuration and implement BGP monitoring"
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
