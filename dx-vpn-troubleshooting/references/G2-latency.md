---
title: "G2 â€” Latency Issues"
description: "Diagnose latency problems on Direct Connect and VPN connections"
status: active
severity: MEDIUM
triggers:
  - "high latency"
  - "slow response"
  - "DX latency"
  - "VPN latency"
  - "round trip time"
owner: devops-agent
objective: "Identify sources of latency and optimize network path performance"
context: "DX latency depends on the physical distance between the customer and the DX location, plus AWS backbone latency. VPN latency includes internet path latency plus encryption overhead. Cross-AZ mount targets, cross-region routing, and MTU mismatches can add latency. Accelerated VPN uses AWS Global Accelerator to reduce internet path latency."
---

## Phase 1 â€” Triage

MUST:
- Measure round-trip latency across the connection (ping, traceroute)
- Check DX location proximity to the customer site
- For VPN: check internet path latency to the VPN endpoint
- Verify MTU settings (fragmentation adds latency)

SHOULD:
- Compare latency to baseline or expected values for the distance
- Check for packet loss which causes retransmissions and perceived latency
- Verify no asymmetric routing is adding extra hops

MAY:
- Use AWS Network Manager for path analysis
- Check for congestion on the DX connection (high utilization)
- Test with different packet sizes to identify MTU issues

## Phase 2 â€” Remediate

MUST:
- If MTU mismatch: align MTU settings end-to-end (DX supports 9001, VPN typically 1400-1500)
- If packet loss: investigate physical layer or congestion issues
- If internet path latency (VPN): consider Accelerated VPN or DX

SHOULD:
- Use the closest DX location to minimize physical distance
- Enable jumbo frames on DX for reduced per-packet overhead
- For VPN: use Accelerated Site-to-Site VPN for AWS Global Accelerator optimization

MAY:
- Implement application-level optimizations (connection pooling, caching)
- Consider AWS Direct Connect SiteLink for site-to-site traffic

## Common Issues

- symptoms: "DX latency higher than expected for the distance"
  diagnosis: "MTU mismatch causing fragmentation, or congestion on the connection."
  resolution: "Align MTU settings. Check connection utilization. Verify no fragmentation."

- symptoms: "VPN latency is high and variable"
  diagnosis: "Internet path congestion or routing changes."
  resolution: "Use Accelerated VPN for consistent low-latency path via AWS Global Accelerator."

- symptoms: "Latency spikes at specific times"
  diagnosis: "Connection congestion during peak hours."
  resolution: "Upgrade connection capacity or implement QoS/traffic shaping."

## Output Format

```yaml
root_cause: "latency_issue â€” <specific_cause>"
evidence:
  - type: latency_measurements
    content: "<RTT measurements>"
  - type: mtu_settings
    content: "<MTU configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix MTU mismatch or reduce congestion"
  long_term: "Optimize network path and implement latency monitoring"
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
