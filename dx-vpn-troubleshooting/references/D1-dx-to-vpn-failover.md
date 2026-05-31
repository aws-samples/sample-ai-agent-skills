---
title: "D1 â€” DX-to-VPN Failover"
description: "Diagnose Direct Connect to VPN failover configuration and behavior"
status: active
severity: HIGH
triggers:
  - "DX failover"
  - "failover to VPN"
  - "DX backup"
  - "VPN backup for DX"
  - "failover not working"
owner: devops-agent
objective: "Verify and fix DX-to-VPN failover configuration for reliable redundancy"
context: "DX-to-VPN failover requires both paths to be active with BGP configured to prefer DX. When DX fails, BGP withdraws DX routes and traffic shifts to VPN. Without proper BGP tuning (AS path prepending, local preference), failover may not work correctly or traffic may prefer VPN over DX."
---

## Phase 1 â€” Triage

MUST:
- Check DX connection and VIF status: `aws directconnect describe-virtual-interfaces`
- Check VPN tunnel status: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].VgwTelemetry'`
- Verify both DX and VPN are advertising routes to the same destinations
- Check VPC route table to see which path is preferred for on-premises CIDRs

SHOULD:
- Verify BGP AS path length for routes over DX vs VPN
- Check BGP local preference on the customer router
- Confirm VPN tunnels are both UP (ready for failover)

MAY:
- Test failover by simulating DX failure (disable BGP on customer router DX interface)
- Measure failover convergence time

## Phase 2 â€” Remediate

MUST:
- Configure AS path prepending on VPN routes to make VPN less preferred than DX
- Ensure both DX and VPN BGP sessions are established and routes are exchanged
- Verify VPC route tables receive routes from both paths

SHOULD:
- Set BGP local preference on customer router: higher for DX, lower for VPN
- Enable BFD on DX for faster failure detection
- Test failover and failback procedures

MAY:
- Implement health checks to trigger failover
- Document failover and failback procedures

## Common Issues

- symptoms: "DX goes down but traffic does not shift to VPN"
  diagnosis: "VPN BGP session not established, or VPN routes not in VPC route table."
  resolution: "Ensure VPN tunnels are UP with BGP established. Enable route propagation for VPN routes."

- symptoms: "Traffic prefers VPN over DX even when DX is healthy"
  diagnosis: "VPN route has shorter AS path or higher local preference than DX route."
  resolution: "Prepend AS path on VPN advertisements. Set higher local preference for DX on customer router."

- symptoms: "Failover works but failback does not"
  diagnosis: "DX BGP session re-establishes but routes are not preferred over VPN."
  resolution: "Verify BGP timers and ensure DX routes are preferred when DX recovers."

## Output Format

```yaml
root_cause: "dx_vpn_failover â€” <specific_cause>"
evidence:
  - type: dx_status
    content: "<DX connection and VIF state>"
  - type: vpn_status
    content: "<VPN tunnel state>"
  - type: route_preference
    content: "<route table showing path preference>"
severity: HIGH
mitigation:
  immediate: "Fix BGP path preference and verify failover readiness"
  long_term: "Implement automated failover testing and monitoring"
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
