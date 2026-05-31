---
title: "C2 â€” Route Table Issues"
description: "Diagnose VPC route table misconfigurations affecting DX and VPN connectivity"
status: active
severity: HIGH
triggers:
  - "route table wrong"
  - "wrong route"
  - "route conflict"
  - "traffic going wrong path"
  - "route not pointing to VGW"
owner: devops-agent
objective: "Identify route table misconfigurations and correct routing for DX/VPN traffic"
context: "VPC route tables control traffic flow. Static routes take precedence over propagated routes for the same CIDR. The most specific route wins. Route table association determines which subnets use which routes. Common issues include missing routes, wrong targets, and conflicts between static and propagated routes."
---

## Phase 1 â€” Triage

MUST:
- List all route tables in the VPC: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`
- Check route table associations (which subnets use which route table)
- Verify routes pointing to VGW or TGW for on-premises CIDRs
- Check for conflicting static routes that override propagated routes

SHOULD:
- Verify the main route table vs custom route tables
- Check if any route has a blackhole state
- Verify NACLs are not blocking traffic on the subnet

MAY:
- Compare route tables across subnets to find inconsistencies
- Check for overlapping CIDRs between VPC and on-premises

## Phase 2 â€” Remediate

MUST:
- Add missing routes to VGW/TGW for on-premises CIDRs
- Remove conflicting static routes that override correct propagated routes
- Fix route table associations so affected subnets use the correct route table

SHOULD:
- Enable route propagation instead of relying on static routes
- Document the route table design and subnet associations
- Remove blackhole routes

MAY:
- Consolidate route tables to reduce complexity
- Implement route table change monitoring via CloudTrail

## Common Issues

- symptoms: "Some subnets can reach on-premises, others cannot"
  diagnosis: "Different route tables associated with different subnets. Some lack VGW/TGW routes."
  resolution: "Check route table associations. Add VGW/TGW routes or enable propagation on all needed route tables."

- symptoms: "Traffic goes to internet gateway instead of VGW"
  diagnosis: "A more specific route (e.g., 0.0.0.0/0 to IGW) is overriding the on-premises route."
  resolution: "Add a more specific route for on-premises CIDR pointing to VGW/TGW."

- symptoms: "Route shows blackhole state"
  diagnosis: "The target (VGW, TGW) has been deleted or detached."
  resolution: "Reattach the VGW/TGW or update the route to point to the correct target."

## Output Format

```yaml
root_cause: "route_table_issue â€” <specific_cause>"
evidence:
  - type: route_tables
    content: "<route table entries and associations>"
  - type: propagation
    content: "<propagation settings>"
severity: HIGH
mitigation:
  immediate: "Fix route table entries and associations"
  long_term: "Document routing design and implement change monitoring"
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
