---
title: "H2 â€” VPN Migration"
description: "Guide for migrating VPN connections between VGW and TGW, or upgrading VPN configurations"
status: active
severity: MEDIUM
triggers:
  - "VPN migration"
  - "migrate VPN to TGW"
  - "VGW to TGW"
  - "upgrade VPN"
  - "VPN reconfiguration"
owner: devops-agent
objective: "Guide VPN migration and minimize downtime during the transition"
context: "Common VPN migrations include: VGW-based VPN to TGW-based VPN (for ECMP, centralized routing), upgrading IKE version, changing encryption parameters, or migrating customer gateway devices. Migrations require careful planning to minimize downtime. AWS supports modifying VPN tunnel options without recreating the connection."
---

## Phase 1 â€” Triage

MUST:
- Document current VPN configuration: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id>`
- Identify migration target (VGW to TGW, IKE upgrade, CGW replacement)
- Check current tunnel options and routing configuration
- Verify the target infrastructure exists (TGW, new CGW)

SHOULD:
- Plan for downtime or parallel operation during migration
- Document current BGP routes and route table entries
- Verify the new configuration is compatible with the customer gateway device

MAY:
- Test the new configuration in a non-production environment first
- Prepare rollback procedures

## Phase 2 â€” Remediate

MUST:
- For VGW to TGW migration: create new VPN connection on TGW, configure CGW, verify connectivity, then delete old VPN
- For IKE upgrade: modify tunnel options (can be done without recreating): `aws ec2 modify-vpn-tunnel-options`
- For CGW replacement: create new CGW, modify VPN connection target: `aws ec2 modify-vpn-connection`

SHOULD:
- Run old and new VPN connections in parallel during migration
- Verify routing works correctly before decommissioning the old connection
- Update route tables to point to the new VPN/TGW

MAY:
- Use AWS VPN CloudWatch metrics to verify traffic flow on the new connection
- Implement canary testing before full cutover

## Common Issues

- symptoms: "VPN migration to TGW causes connectivity loss"
  diagnosis: "Route tables not updated to point to TGW, or TGW route table missing routes."
  resolution: "Update VPC route tables to point to TGW. Configure TGW route table associations and propagations."

- symptoms: "Tunnel options modification causes tunnel to drop"
  diagnosis: "Expected behavior. Tunnel renegotiates with new parameters."
  resolution: "Modify one tunnel at a time. Wait for it to re-establish before modifying the second."

- symptoms: "New CGW cannot establish tunnel"
  diagnosis: "New CGW IP not matching the customer-gateway-id, or IKE parameters mismatch."
  resolution: "Create a new customer-gateway with the correct IP. Download new VPN config file."

## Output Format

```yaml
root_cause: "vpn_migration â€” <specific_cause>"
evidence:
  - type: current_config
    content: "<current VPN configuration>"
  - type: target_config
    content: "<target VPN configuration>"
severity: MEDIUM
mitigation:
  immediate: "Complete migration steps and verify connectivity"
  long_term: "Document new configuration and update runbooks"
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
