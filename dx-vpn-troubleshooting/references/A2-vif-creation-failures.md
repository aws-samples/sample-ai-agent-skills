---
title: "A2 â€” VIF Creation Failures"
description: "Diagnose Virtual Interface creation and provisioning failures"
status: active
severity: HIGH
triggers:
  - "VIF creation failed"
  - "virtual interface failed"
  - "VIF stuck in confirming"
  - "VLAN conflict"
owner: devops-agent
objective: "Identify why VIF creation is failing and successfully provision the virtual interface"
context: "VIF creation can fail due to VLAN ID conflicts, incorrect owner account, BGP ASN issues, DX connection not in available state, or exceeding VIF limits. Private VIFs connect to VGW, Public VIFs to AWS public services, Transit VIFs to TGW via DX Gateway."
---

## Phase 1 â€” Triage

MUST:
- Check existing VIFs on the connection: `aws directconnect describe-virtual-interfaces --connection-id <dxcon-id>`
- Verify connection is available: `aws directconnect describe-connections --connection-id <dxcon-id> --query 'connections[].connectionState'`
- Check VIF state: `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id> --query 'virtualInterfaces[].virtualInterfaceState'`
- Verify VLAN ID is not already in use on the connection

SHOULD:
- Check if the VIF owner account has accepted the VIF (for hosted VIFs)
- Verify the VGW or DX Gateway exists and is in the correct state
- Check VIF limits per connection

MAY:
- Check CloudTrail for CreateVirtualInterface API errors
- Verify BGP ASN is valid and not conflicting

## Phase 2 â€” Remediate

MUST:
- Use a unique VLAN ID not already assigned on the connection
- Ensure the DX connection is in "available" state before creating VIFs
- For hosted VIFs, the owner account must confirm/accept the VIF

SHOULD:
- Verify the VGW is attached to the target VPC (for Private VIF)
- Verify the DX Gateway exists (for Transit VIF)
- Use a valid private ASN (64512-65534) for the customer side

MAY:
- Delete and recreate the VIF if stuck in a bad state

## Common Issues

- symptoms: "VIF stuck in 'confirming' state"
  diagnosis: "The VIF owner account has not accepted the hosted VIF."
  resolution: "The owner account must call confirm-virtual-interface to accept it."

- symptoms: "VIF creation fails with VLAN conflict"
  diagnosis: "The VLAN ID is already in use by another VIF on the same connection."
  resolution: "Choose a different VLAN ID (1-4094) not already assigned."

- symptoms: "Private VIF fails to associate with VGW"
  diagnosis: "VGW does not exist, is in wrong region, or is already associated with another VIF."
  resolution: "Verify VGW exists, is in the same region, and is not at its association limit."

## Output Format

```yaml
root_cause: "vif_creation_failure â€” <specific_cause>"
evidence:
  - type: vif_state
    content: "<VIF configuration and state>"
  - type: connection_state
    content: "<DX connection state>"
severity: HIGH
mitigation:
  immediate: "Fix VIF configuration and retry creation"
  long_term: "Document VLAN allocation and VIF provisioning procedures"
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
