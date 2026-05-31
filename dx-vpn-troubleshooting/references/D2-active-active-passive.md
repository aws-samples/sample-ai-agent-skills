---
title: "D2 â€” Active-Active vs Active-Passive Configuration"
description: "Diagnose and configure active-active or active-passive DX/VPN architectures"
status: active
severity: MEDIUM
triggers:
  - "active-active"
  - "active-passive"
  - "load balancing DX"
  - "dual DX"
  - "redundant connections"
owner: devops-agent
objective: "Verify the correct active-active or active-passive configuration for DX/VPN redundancy"
context: "Active-active uses both connections simultaneously for load balancing (requires ECMP or traffic engineering). Active-passive keeps one connection as standby. The choice depends on throughput needs, cost tolerance, and failover requirements. DX supports active-active with multiple VIFs or LAG. VPN supports ECMP on Transit Gateway."
---

## Phase 1 â€” Triage

MUST:
- Identify all DX connections: `aws directconnect describe-connections`
- Identify all VPN connections: `aws ec2 describe-vpn-connections`
- Check which connections are actively carrying traffic (CloudWatch metrics)
- Determine the intended architecture (active-active or active-passive)

SHOULD:
- Check BGP attributes to understand path preference
- Verify ECMP is enabled on Transit Gateway (if using TGW)
- Check if traffic is balanced across connections

MAY:
- Review the network architecture documentation
- Check for asymmetric routing between active connections

## Phase 2 â€” Remediate

MUST:
- For active-active: ensure both connections advertise the same prefixes with equal AS path length
- For active-passive: ensure the passive connection has longer AS path or lower local preference
- Verify failover works when the active connection fails

SHOULD:
- For active-active with TGW: enable ECMP on the Transit Gateway
- For active-passive: test failover and measure convergence time
- Monitor both connections for health

MAY:
- Implement traffic engineering for specific prefix routing over specific connections
- Document the redundancy architecture and failover procedures

## Common Issues

- symptoms: "Both DX connections active but traffic only uses one"
  diagnosis: "BGP attributes make one path preferred. Not true active-active."
  resolution: "Equalize AS path length and local preference for both connections."

- symptoms: "Active-passive failover takes too long"
  diagnosis: "BGP hold timer too long. BFD not enabled."
  resolution: "Enable BFD for sub-second failure detection. Reduce BGP hold timer."

- symptoms: "After failover, traffic doesn't return to primary when restored"
  diagnosis: "BGP attributes are equal, so traffic stays on the secondary path."
  resolution: "Ensure primary path has shorter AS path or higher local preference for failback."

## Output Format

```yaml
root_cause: "active_active_passive â€” <specific_cause>"
evidence:
  - type: connections
    content: "<all DX and VPN connections>"
  - type: traffic_distribution
    content: "<CloudWatch metrics showing traffic per connection>"
severity: MEDIUM
mitigation:
  immediate: "Fix BGP attributes for desired active-active or active-passive behavior"
  long_term: "Document architecture and implement automated failover testing"
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
