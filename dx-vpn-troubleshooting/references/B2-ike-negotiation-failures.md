---
title: "B2 â€” IKE Negotiation Failures"
description: "Diagnose IKE Phase 1 and Phase 2 negotiation failures for Site-to-Site VPN"
status: active
severity: HIGH
triggers:
  - "IKE failed"
  - "Phase 1 failed"
  - "Phase 2 failed"
  - "IKE negotiation"
  - "ISAKMP"
  - "no proposal chosen"
owner: devops-agent
objective: "Identify IKE negotiation mismatches and establish the IPsec tunnel"
context: "IKE negotiation has two phases. Phase 1 (IKE SA) establishes a secure channel using encryption, hashing, DH group, and authentication. Phase 2 (IPsec SA) negotiates the IPsec tunnel parameters. Mismatches in any parameter cause negotiation failure."
---

## Phase 1 â€” Triage

MUST:
- Check VPN tunnel options: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].Options.TunnelOptions'`
- Verify IKE version (IKEv1 or IKEv2) matches on both sides
- Check VgwTelemetry for status detail messages: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].VgwTelemetry[].statusMessage'`
- Verify pre-shared key matches exactly (case-sensitive)

SHOULD:
- Compare Phase 1 parameters: encryption algorithm, integrity algorithm, DH group, lifetime
- Compare Phase 2 parameters: encryption algorithm, integrity algorithm, PFS DH group, lifetime
- Check customer gateway device logs for specific IKE error messages

MAY:
- Enable IKE debug logging on the customer gateway device
- Check for NAT-Traversal (NAT-T) requirements if CGW is behind NAT

## Phase 2 â€” Remediate

MUST:
- Align IKE version on both sides (IKEv2 recommended)
- Match Phase 1 parameters: encryption (AES-256), integrity (SHA-256), DH group (14 or higher)
- Match Phase 2 parameters: encryption (AES-256), integrity (SHA-256), PFS group
- Verify pre-shared key is identical on both sides

SHOULD:
- Use IKEv2 for better performance and reliability
- Set appropriate IKE and IPsec lifetimes
- Enable NAT-T if the customer gateway is behind a NAT device

MAY:
- Modify VPN tunnel options via AWS API to match customer gateway capabilities
- Use certificate-based authentication instead of pre-shared keys

## Common Issues

- symptoms: "Status message: 'IPSEC IS DOWN'"
  diagnosis: "Phase 1 or Phase 2 negotiation failed. Parameter mismatch between AWS and customer gateway."
  resolution: "Download VPN config file and compare all IKE/IPsec parameters with customer gateway config."

- symptoms: "'no proposal chosen' in customer gateway logs"
  diagnosis: "No matching encryption/integrity/DH group proposal between the two sides."
  resolution: "Align Phase 1 and Phase 2 proposals. Use AWS-supported algorithms."

- symptoms: "Phase 1 succeeds but Phase 2 fails"
  diagnosis: "IPsec SA parameters mismatch. Often PFS group or Phase 2 encryption mismatch."
  resolution: "Check Phase 2 encryption, integrity, and PFS DH group settings."

## Output Format

```yaml
root_cause: "ike_negotiation_failure â€” <specific_cause>"
evidence:
  - type: tunnel_options
    content: "<IKE/IPsec parameters configured>"
  - type: vgw_telemetry
    content: "<status message from AWS>"
severity: HIGH
mitigation:
  immediate: "Align IKE/IPsec parameters on both sides"
  long_term: "Document IKE parameters and use IKEv2 with strong algorithms"
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
