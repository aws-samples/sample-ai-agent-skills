---
title: "F2 â€” VPN Encryption Issues"
description: "Diagnose IPsec encryption and security configuration for Site-to-Site VPN"
status: active
severity: HIGH
triggers:
  - "VPN encryption"
  - "IPsec security"
  - "weak cipher"
  - "VPN security"
  - "pre-shared key"
owner: devops-agent
objective: "Identify VPN encryption misconfigurations and ensure strong security posture"
context: "Site-to-Site VPN uses IPsec for encryption. AWS supports IKEv1 and IKEv2 with various encryption algorithms (AES-128, AES-256), integrity algorithms (SHA-1, SHA-256), and DH groups. Weak algorithms should be avoided. Pre-shared keys must be managed securely. Certificate-based authentication is available as an alternative."
---

## Phase 1 â€” Triage

MUST:
- Check VPN tunnel options: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].Options.TunnelOptions'`
- Verify IKE version in use (IKEv2 recommended)
- Check Phase 1 and Phase 2 encryption algorithms
- Verify DH group strength (Group 14 or higher recommended)

SHOULD:
- Check if pre-shared key is using a strong, random value
- Verify integrity algorithm (SHA-256 or higher recommended)
- Check if Perfect Forward Secrecy (PFS) is enabled for Phase 2

MAY:
- Check if certificate-based authentication is configured
- Review VPN logging for security events

## Phase 2 â€” Remediate

MUST:
- Use IKEv2 instead of IKEv1 for better security and performance
- Use AES-256 for encryption in both Phase 1 and Phase 2
- Use SHA-256 or higher for integrity
- Use DH Group 14 or higher

SHOULD:
- Enable PFS for Phase 2 with a strong DH group
- Use a strong, random pre-shared key (at least 32 characters)
- Consider certificate-based authentication for stronger security

MAY:
- Enable VPN logging for security monitoring
- Implement pre-shared key rotation procedures
- Modify tunnel options: `aws ec2 modify-vpn-tunnel-options --vpn-connection-id <vpn-id> --vpn-tunnel-outside-ip-address <ip> --tunnel-options <options>`

## Common Issues

- symptoms: "Security audit flags weak VPN encryption"
  diagnosis: "Using deprecated algorithms (DES, 3DES, SHA-1, DH Group 2)."
  resolution: "Update tunnel options to use AES-256, SHA-256, and DH Group 14+."

- symptoms: "VPN tunnel fails after changing encryption settings"
  diagnosis: "Customer gateway does not support the new algorithms."
  resolution: "Verify customer gateway supports the chosen algorithms. Align both sides."

- symptoms: "Pre-shared key compromised"
  diagnosis: "PSK needs rotation."
  resolution: "Modify VPN tunnel options with a new PSK. Update customer gateway configuration."

## Output Format

```yaml
root_cause: "vpn_encryption â€” <specific_cause>"
evidence:
  - type: tunnel_options
    content: "<IKE/IPsec encryption parameters>"
  - type: ike_version
    content: "<IKE version in use>"
severity: HIGH
mitigation:
  immediate: "Update encryption parameters to strong algorithms"
  long_term: "Implement encryption policy and key rotation procedures"
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
