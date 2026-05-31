---
title: "F1 â€” MACsec Encryption"
description: "Diagnose MACsec (802.1AE) encryption issues on Direct Connect"
status: active
severity: HIGH
triggers:
  - "MACsec"
  - "802.1AE"
  - "Layer 2 encryption"
  - "DX encryption"
  - "MACsec key"
owner: devops-agent
objective: "Identify MACsec configuration issues and establish encrypted DX connectivity"
context: "MACsec provides Layer 2 encryption on Direct Connect. It is only available on dedicated connections at 10 Gbps and 100 Gbps. Both the AWS side and customer router must support and configure MACsec. A Connection Key Name (CKN) and Connectivity Association Key (CAK) must match on both sides."
---

## Phase 1 â€” Triage

MUST:
- Verify connection supports MACsec: `aws directconnect describe-connections --connection-id <dxcon-id> --query 'connections[].macSecCapable'`
- Check MACsec keys: `aws directconnect describe-connections --connection-id <dxcon-id> --query 'connections[].macSecKeys'`
- Verify connection speed is 10 Gbps or 100 Gbps (MACsec requirement)
- Check if MACsec encryption mode is configured: `aws directconnect describe-connections --connection-id <dxcon-id> --query 'connections[].encryptionMode'`

SHOULD:
- Verify the CKN/CAK pair is active and not in "failed" state
- Check customer router MACsec configuration matches AWS-side parameters
- Verify the connection is a dedicated connection (not hosted)

MAY:
- Check customer router MACsec session status
- Verify MACsec cipher suite compatibility

## Phase 2 â€” Remediate

MUST:
- Ensure the connection is a dedicated 10 Gbps or 100 Gbps connection
- Associate a MACsec CKN/CAK key: `aws directconnect associate-mac-sec-key --connection-id <dxcon-id> --secret-arn <secret-arn>`
- Configure matching CKN/CAK on the customer router
- Set encryption mode to "should_encrypt" or "must_encrypt"

SHOULD:
- Store the CAK in AWS Secrets Manager
- Use "must_encrypt" mode for strict encryption enforcement
- Test connectivity after enabling MACsec

MAY:
- Rotate MACsec keys periodically
- Monitor MACsec session status

## Common Issues

- symptoms: "MACsec key shows 'failed' state"
  diagnosis: "CKN/CAK mismatch between AWS and customer router, or unsupported cipher suite."
  resolution: "Verify CKN/CAK matches exactly on both sides. Check cipher suite compatibility."

- symptoms: "Connection drops after enabling MACsec"
  diagnosis: "MACsec negotiation failing. Customer router may not support the required cipher suite."
  resolution: "Verify customer router MACsec support. Set encryption mode to 'should_encrypt' to allow fallback."

- symptoms: "Cannot enable MACsec on the connection"
  diagnosis: "Connection is hosted, sub-10 Gbps, or not MACsec-capable."
  resolution: "MACsec requires a dedicated 10 Gbps or 100 Gbps connection. Use VPN over DX for lower speeds."

## Output Format

```yaml
root_cause: "macsec_encryption â€” <specific_cause>"
evidence:
  - type: macsec_capability
    content: "<connection MACsec capability and keys>"
  - type: encryption_mode
    content: "<encryption mode setting>"
severity: HIGH
mitigation:
  immediate: "Fix MACsec key configuration or encryption mode"
  long_term: "Implement MACsec key rotation and monitoring"
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
