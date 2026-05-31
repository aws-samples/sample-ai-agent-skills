---
title: "B4 â€” VPN Throughput Limits"
description: "Diagnose VPN throughput bottlenecks and per-tunnel bandwidth limits"
status: active
severity: MEDIUM
triggers:
  - "VPN slow"
  - "VPN throughput"
  - "VPN bandwidth"
  - "1.25 Gbps limit"
  - "VPN performance"
owner: devops-agent
objective: "Identify VPN throughput bottlenecks and optimize bandwidth utilization"
context: "Each VPN tunnel has a hard limit of 1.25 Gbps and 140,000 packets per second. This cannot be increased. For higher throughput, use multiple VPN connections with ECMP on Transit Gateway, or use Direct Connect. Small packet workloads may hit the PPS limit before the bandwidth limit."
---

## Phase 1 â€” Triage

MUST:
- Check VPN tunnel throughput metrics: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelDataIn --dimensions Name=VpnId,Value=<vpn-id> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Check TunnelDataOut similarly
- Verify both tunnels are active and carrying traffic
- Determine if the workload is bandwidth-limited or PPS-limited

SHOULD:
- Check if traffic is balanced across both tunnels (ECMP)
- Verify the customer gateway device is not the bottleneck (CPU, memory)
- Check for packet loss on the VPN tunnels

MAY:
- Measure latency across the VPN tunnel
- Check if encryption overhead is significant for the workload

## Phase 2 â€” Remediate

MUST:
- Confirm the 1.25 Gbps per-tunnel limit is the bottleneck (not something else)
- If more throughput is needed, plan for multiple VPN connections with ECMP on TGW

SHOULD:
- Enable ECMP on Transit Gateway for load balancing across multiple VPN connections
- Consider Direct Connect for workloads requiring more than 2.5 Gbps aggregate VPN throughput
- Optimize packet sizes to avoid hitting PPS limits

MAY:
- Use VPN acceleration (Accelerated Site-to-Site VPN) for improved performance
- Implement traffic shaping on the customer gateway

## Common Issues

- symptoms: "VPN throughput plateaus at ~1.25 Gbps"
  diagnosis: "Hitting the per-tunnel AWS hard limit of 1.25 Gbps."
  resolution: "Use multiple VPN connections with ECMP on Transit Gateway, or migrate to Direct Connect."

- symptoms: "Throughput is well below 1.25 Gbps but still slow"
  diagnosis: "Customer gateway CPU bottleneck, internet path congestion, or small packet overhead."
  resolution: "Check CGW device resources. Consider Accelerated VPN. Optimize packet sizes."

- symptoms: "Traffic only flows through one tunnel"
  diagnosis: "ECMP not configured or customer gateway not load-balancing across tunnels."
  resolution: "Configure ECMP on the customer gateway to distribute traffic across both tunnels."

## Output Format

```yaml
root_cause: "vpn_throughput_limit â€” <specific_cause>"
evidence:
  - type: tunnel_metrics
    content: "<TunnelDataIn/Out metrics>"
  - type: tunnel_state
    content: "<both tunnel states>"
severity: MEDIUM
mitigation:
  immediate: "Optimize tunnel usage or add VPN connections with ECMP"
  long_term: "Evaluate Direct Connect for high-throughput requirements"
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
