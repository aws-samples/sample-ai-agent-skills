---
title: "H1 â€” DX Setup and Provisioning"
description: "Guide for Direct Connect setup, provisioning, and common provisioning issues"
status: active
severity: MEDIUM
triggers:
  - "DX setup"
  - "DX provisioning"
  - "new Direct Connect"
  - "LOA-CFA"
  - "cross-connect"
owner: devops-agent
objective: "Guide through DX provisioning steps and troubleshoot setup issues"
context: "DX provisioning involves: creating a connection in the console, downloading the LOA-CFA, providing it to the colocation provider for cross-connect, waiting for AWS to detect the physical link, creating VIFs, and configuring BGP. The process can take days to weeks depending on the colocation provider. Hosted connections are provisioned by AWS Direct Connect Partners."
---

## Phase 1 â€” Triage

MUST:
- Check connection state: `aws directconnect describe-connections --connection-id <dxcon-id>`
- Verify the connection type (dedicated vs hosted)
- Check if LOA-CFA has been downloaded and provided to the colocation provider
- Verify the connection location matches the customer's colocation facility

SHOULD:
- Check if the colocation provider has completed the cross-connect
- Verify the connection speed matches requirements
- Check if a DX Gateway is needed for multi-region access

MAY:
- Verify the AWS account has the necessary DX service quotas
- Check for any pending DX connection requests

## Phase 2 â€” Remediate

MUST:
- For "ordering" state: download LOA-CFA and provide to colocation provider
- For "pending" state: wait for AWS to detect the physical link (can take 72 hours)
- For "available" state: proceed to create VIFs and configure BGP
- For "down" state: verify physical cross-connect with colocation provider

SHOULD:
- Plan VIF types needed (Private, Public, Transit) before provisioning
- Prepare BGP configuration for the customer router
- Document VLAN IDs and BGP ASN allocation

MAY:
- Request a dedicated connection if hosted connection limits are insufficient
- Set up monitoring before going live

## Common Issues

- symptoms: "Connection stuck in 'ordering' state"
  diagnosis: "LOA-CFA not provided to colocation provider, or cross-connect not completed."
  resolution: "Download LOA-CFA from console. Provide to colocation provider. Wait for cross-connect completion."

- symptoms: "Connection stuck in 'pending' state"
  diagnosis: "AWS has not yet detected the physical link. Can take up to 72 hours."
  resolution: "Wait for AWS detection. If over 72 hours, verify cross-connect with colocation provider."

- symptoms: "Connection goes to 'available' but VIF creation fails"
  diagnosis: "Physical layer is up but VIF configuration has issues."
  resolution: "See A2 runbook for VIF creation troubleshooting."

## Output Format

```yaml
root_cause: "dx_provisioning â€” <specific_cause>"
evidence:
  - type: connection_state
    content: "<connection state and details>"
  - type: provisioning_step
    content: "<current step in provisioning process>"
severity: MEDIUM
mitigation:
  immediate: "Complete the current provisioning step"
  long_term: "Document provisioning procedures and timelines"
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
