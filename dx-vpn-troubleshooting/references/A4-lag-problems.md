---
title: "A4 â€” LAG Problems"
description: "Diagnose Link Aggregation Group configuration and member connection issues"
status: active
severity: HIGH
triggers:
  - "LAG down"
  - "LAG degraded"
  - "link aggregation"
  - "LAG member down"
  - "minimum links"
owner: devops-agent
objective: "Identify LAG configuration issues and restore full LAG capacity"
context: "LAG bundles multiple DX connections for higher bandwidth and redundancy. All members must have the same speed and terminate at the same location. The minimumLinks setting determines how many members must be active for the LAG to remain operational."
---

## Phase 1 â€” Triage

MUST:
- Check LAG status: `aws directconnect describe-lags --lag-id <lag-id>`
- Check LAG member connections: look at `connections` array in LAG output
- Verify all members have the same bandwidth and location
- Check minimumLinks setting vs active member count

SHOULD:
- Check individual member connection states: `aws directconnect describe-connections --connection-id <member-dxcon-id>`
- Verify LACP is configured on the customer router
- Check CloudWatch metrics for each member connection

MAY:
- Check for recent LAG configuration changes in CloudTrail
- Verify the colocation provider has all cross-connects active

## Phase 2 â€” Remediate

MUST:
- Ensure all LAG members are in "available" state
- Fix any down member connections (see A1 runbook)
- Verify LACP negotiation is succeeding on the customer router

SHOULD:
- Adjust minimumLinks if too aggressive for current member count
- Replace failed member connections with new ones of the same speed

MAY:
- Remove and re-add problematic member connections
- Contact colocation provider for physical layer issues on specific members

## Common Issues

- symptoms: "LAG shows fewer active members than expected"
  diagnosis: "One or more member connections are down. Physical layer or provisioning issue."
  resolution: "Check each member connection state. Fix down connections per A1 runbook."

- symptoms: "LAG creation fails"
  diagnosis: "Member connections have different speeds or are at different DX locations."
  resolution: "Ensure all connections have the same bandwidth and terminate at the same location."

- symptoms: "LAG is down despite some members being up"
  diagnosis: "Active member count is below the minimumLinks threshold."
  resolution: "Restore failed members or lower the minimumLinks setting if appropriate."

## Output Format

```yaml
root_cause: "lag_problem â€” <specific_cause>"
evidence:
  - type: lag_state
    content: "<LAG configuration and member states>"
  - type: member_connections
    content: "<individual connection states>"
severity: HIGH
mitigation:
  immediate: "Restore failed LAG members or adjust minimumLinks"
  long_term: "Monitor LAG member health and maintain spare capacity"
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
