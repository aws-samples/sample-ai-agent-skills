---
title: "G1 â€” Bandwidth Issues"
description: "Diagnose bandwidth bottlenecks on Direct Connect and VPN connections"
status: active
severity: MEDIUM
triggers:
  - "bandwidth low"
  - "throughput low"
  - "slow transfer"
  - "DX bandwidth"
  - "not getting full speed"
owner: devops-agent
objective: "Identify bandwidth bottlenecks and optimize throughput on DX/VPN connections"
context: "DX bandwidth depends on the connection speed (1 Gbps, 10 Gbps, 100 Gbps for dedicated; 50 Mbps to 10 Gbps for hosted). VPN is limited to 1.25 Gbps per tunnel. Bottlenecks can occur at the physical layer, VIF level, VPN tunnel, or application layer. CloudWatch metrics show actual throughput."
---

## Phase 1 â€” Triage

MUST:
- Check DX connection bandwidth: `aws directconnect describe-connections --connection-id <dxcon-id> --query 'connections[].bandwidth'`
- Check DX throughput metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DX --metric-name ConnectionBpsIngress --dimensions Name=ConnectionId,Value=<dxcon-id> --start-time <time> --end-time <time> --period 300 --statistics Average`
- Check ConnectionBpsEgress similarly
- For VPN: check TunnelDataIn and TunnelDataOut metrics

SHOULD:
- Compare actual throughput to connection capacity
- Check for packet loss or errors on the connection
- Verify the bottleneck is the DX/VPN link and not the application or instance

MAY:
- Run iperf or similar throughput test across the connection
- Check for TCP window size limitations
- Verify jumbo frames are configured end-to-end (DX supports 9001 MTU)

## Phase 2 â€” Remediate

MUST:
- If DX is at capacity: upgrade connection speed or add connections (LAG)
- If VPN is at 1.25 Gbps: add VPN connections with ECMP on TGW
- If neither is at capacity: investigate application-layer bottleneck

SHOULD:
- Enable jumbo frames (MTU 9001) on DX for better throughput efficiency
- Optimize TCP window size for high-latency links
- Use multiple flows for parallel transfers

MAY:
- Implement traffic shaping to prioritize critical traffic
- Consider AWS Direct Connect SiteLink for optimized routing

## Common Issues

- symptoms: "DX throughput well below connection speed"
  diagnosis: "Single TCP flow limited by TCP window size and latency. Or MTU mismatch causing fragmentation."
  resolution: "Use multiple parallel flows. Enable jumbo frames. Tune TCP window size."

- symptoms: "VPN throughput capped at 1.25 Gbps"
  diagnosis: "Hitting the per-tunnel AWS hard limit."
  resolution: "Add VPN connections with ECMP on Transit Gateway for higher aggregate throughput."

- symptoms: "Throughput drops at certain times"
  diagnosis: "Shared hosted connection with other tenants, or internet congestion for VPN."
  resolution: "For DX: upgrade to dedicated connection. For VPN: consider Accelerated VPN or DX."

## Output Format

```yaml
root_cause: "bandwidth_issue â€” <specific_cause>"
evidence:
  - type: connection_bandwidth
    content: "<connection speed and actual throughput>"
  - type: cloudwatch_metrics
    content: "<throughput metrics>"
severity: MEDIUM
mitigation:
  immediate: "Optimize throughput or add capacity"
  long_term: "Right-size connections and implement throughput monitoring"
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
