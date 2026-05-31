---
title: "C3 â€” Asymmetric Routing"
description: "Diagnose asymmetric routing between Direct Connect and VPN paths"
status: active
severity: MEDIUM
triggers:
  - "asymmetric routing"
  - "traffic taking different path"
  - "DX one way VPN other"
  - "return path different"
owner: devops-agent
objective: "Identify and resolve asymmetric routing where traffic takes different paths in each direction"
context: "Asymmetric routing occurs when traffic from on-premises to AWS takes one path (e.g., DX) but return traffic takes a different path (e.g., VPN). This can cause issues with stateful firewalls, NAT devices, and connection tracking. Common when both DX and VPN are active without proper BGP tuning."
---

## Phase 1 â€” Triage

MUST:
- Check routes advertised over DX: `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id>`
- Check routes advertised over VPN: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].Routes'`
- Compare VPC route table entries for on-premises CIDRs (which target: VGW via DX or VGW via VPN?)
- Check BGP AS path length for routes received over both paths

SHOULD:
- Verify BGP local preference and MED values on the customer router
- Check if the customer router prefers one path for outbound but AWS prefers another for return
- Verify no static routes are overriding BGP-learned routes

MAY:
- Perform traceroute from both directions to confirm asymmetry
- Check stateful firewall logs for dropped return traffic

## Phase 2 â€” Remediate

MUST:
- Align BGP path preference on both sides (AWS and customer)
- Use AS path prepending to make the less-preferred path longer
- Ensure the same path is preferred in both directions

SHOULD:
- Set BGP local preference on the customer router to prefer DX over VPN
- Use AS path prepending on VPN advertisements to make VPN less preferred
- Verify with traceroute from both directions after changes

MAY:
- Configure stateful firewalls to handle asymmetric routing if it cannot be eliminated
- Implement BFD for faster convergence on the preferred path

## Common Issues

- symptoms: "Traffic from on-premises goes over DX but return traffic comes over VPN"
  diagnosis: "AWS route table prefers VPN path (shorter AS path or static route) for return traffic."
  resolution: "Prepend AS path on VPN routes or add static routes preferring DX in VPC route tables."

- symptoms: "Stateful firewall drops return traffic"
  diagnosis: "Asymmetric routing causes return packets to arrive on a different interface than the outbound path."
  resolution: "Fix routing symmetry or configure firewall to handle asymmetric flows."

- symptoms: "Intermittent connectivity when both DX and VPN are active"
  diagnosis: "Traffic flapping between paths due to equal-cost routes."
  resolution: "Ensure clear path preference using AS path prepending or local preference."

## Output Format

```yaml
root_cause: "asymmetric_routing â€” <specific_cause>"
evidence:
  - type: dx_routes
    content: "<routes over DX with AS path>"
  - type: vpn_routes
    content: "<routes over VPN with AS path>"
  - type: route_tables
    content: "<VPC route table entries>"
severity: MEDIUM
mitigation:
  immediate: "Align BGP path preference to eliminate asymmetry"
  long_term: "Document routing policy and implement path monitoring"
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
