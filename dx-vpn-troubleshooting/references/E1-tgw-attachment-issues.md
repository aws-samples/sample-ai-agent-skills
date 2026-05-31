---
title: "E1 â€” Transit Gateway Attachment Issues"
description: "Diagnose Transit Gateway attachment failures for DX and VPN"
status: active
severity: HIGH
triggers:
  - "TGW attachment failed"
  - "Transit Gateway attachment"
  - "TGW not connected"
  - "transit VIF"
  - "VPN TGW attachment"
owner: devops-agent
objective: "Identify and resolve Transit Gateway attachment issues for DX and VPN connections"
context: "Transit Gateway centralizes network connectivity. DX connects via Transit VIF through a DX Gateway. VPN connects directly to TGW. Attachment issues can be caused by missing DX Gateway association, incorrect TGW route table, wrong attachment state, or resource share issues for cross-account."
---

## Phase 1 â€” Triage

MUST:
- Check TGW attachments: `aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=<tgw-id>`
- Check attachment state (should be "available")
- For DX: verify DX Gateway association: `aws directconnect describe-direct-connect-gateway-associations --direct-connect-gateway-id <dxgw-id>`
- For VPN: verify VPN is attached to TGW: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].TransitGatewayId'`

SHOULD:
- Check TGW route table associations: `aws ec2 describe-transit-gateway-route-tables --transit-gateway-id <tgw-id>`
- Verify the Transit VIF is in "available" state
- Check for cross-account resource sharing if TGW is in a different account

MAY:
- Check CloudTrail for TGW attachment API errors
- Verify TGW has available attachment capacity

## Phase 2 â€” Remediate

MUST:
- For DX: create Transit VIF and associate DX Gateway with TGW
- For VPN: create VPN connection with TGW as the target (not VGW)
- Ensure attachment reaches "available" state

SHOULD:
- Associate the attachment with the correct TGW route table
- Configure route propagation on the TGW route table
- For cross-account: accept the resource share and attachment

MAY:
- Tag attachments for identification
- Implement TGW attachment monitoring

## Common Issues

- symptoms: "Transit VIF created but TGW attachment not appearing"
  diagnosis: "DX Gateway not associated with the Transit Gateway."
  resolution: "Create a DX Gateway association with the TGW: `aws directconnect create-direct-connect-gateway-association`"

- symptoms: "VPN attachment stuck in 'pending'"
  diagnosis: "Cross-account TGW requires resource share acceptance."
  resolution: "Accept the RAM resource share in the VPN account."

- symptoms: "Attachment is 'available' but no traffic flows"
  diagnosis: "TGW route table not associated or routes not propagated."
  resolution: "Associate the attachment with a TGW route table and enable route propagation."

## Output Format

```yaml
root_cause: "tgw_attachment_issue â€” <specific_cause>"
evidence:
  - type: tgw_attachments
    content: "<attachment state and configuration>"
  - type: dx_gateway_association
    content: "<DX Gateway to TGW association>"
severity: HIGH
mitigation:
  immediate: "Fix attachment configuration and verify state"
  long_term: "Implement TGW attachment monitoring and documentation"
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
