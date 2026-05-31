---
title: "E2 â€” Transit Gateway Route Table Issues"
description: "Diagnose Transit Gateway route table configuration and propagation problems"
status: active
severity: HIGH
triggers:
  - "TGW routes missing"
  - "TGW route table"
  - "Transit Gateway routing"
  - "TGW propagation"
  - "traffic not flowing through TGW"
owner: devops-agent
objective: "Identify TGW route table misconfigurations and restore correct routing"
context: "Transit Gateway uses route tables to control traffic flow between attachments (VPCs, VPNs, DX). Each attachment is associated with one route table and can propagate routes to multiple route tables. Missing associations, disabled propagation, or incorrect static routes cause connectivity failures."
---

## Phase 1 â€” Triage

MUST:
- List TGW route tables: `aws ec2 describe-transit-gateway-route-tables --transit-gateway-id <tgw-id>`
- Check route table associations: `aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <tgw-rtb-id>`
- Check route table propagations: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <tgw-rtb-id>`
- Search routes: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=type,Values=propagated`

SHOULD:
- Check for blackhole routes in the TGW route table
- Verify static routes are pointing to the correct attachments
- Check if the default route table is being used or custom route tables

MAY:
- Compare route tables across different TGW route tables
- Check for overlapping CIDRs from different attachments

## Phase 2 â€” Remediate

MUST:
- Associate each attachment with the correct TGW route table
- Enable route propagation for attachments that should share routes
- Remove blackhole routes or fix the target attachment

SHOULD:
- Use separate TGW route tables for network segmentation
- Add static routes for specific traffic steering
- Document the TGW route table design

MAY:
- Implement TGW route table change monitoring
- Use TGW route table tags for organization

## Common Issues

- symptoms: "VPC cannot reach on-premises through TGW"
  diagnosis: "VPC attachment not associated with a route table that has on-premises routes."
  resolution: "Associate the VPC attachment with the correct TGW route table. Enable DX/VPN route propagation."

- symptoms: "On-premises can reach one VPC but not another"
  diagnosis: "Target VPC routes not propagated to the DX/VPN route table."
  resolution: "Enable propagation from the target VPC attachment to the DX/VPN TGW route table."

- symptoms: "TGW route shows blackhole"
  diagnosis: "The target attachment was deleted or is in a failed state."
  resolution: "Remove the blackhole route or fix the target attachment."

## Output Format

```yaml
root_cause: "tgw_route_table â€” <specific_cause>"
evidence:
  - type: tgw_routes
    content: "<TGW route table entries>"
  - type: associations
    content: "<attachment associations>"
  - type: propagations
    content: "<route propagation settings>"
severity: HIGH
mitigation:
  immediate: "Fix route table associations and propagation"
  long_term: "Document TGW routing design and implement monitoring"
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
