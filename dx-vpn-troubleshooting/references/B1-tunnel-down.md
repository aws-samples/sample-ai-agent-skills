---
title: "B1 â€” VPN Tunnel Down"
description: "Diagnose Site-to-Site VPN tunnel down state"
status: active
severity: CRITICAL
triggers:
  - "VPN tunnel down"
  - "tunnel state DOWN"
  - "TunnelState 0"
  - "VPN not connecting"
  - "no VPN connectivity"
owner: devops-agent
objective: "Identify why the VPN tunnel is down and restore connectivity"
context: "Each VPN connection has two tunnels. Both should be UP. Tunnel down can be caused by IKE negotiation failure, incorrect pre-shared key, customer gateway misconfiguration, network ACL blocking, or AWS maintenance. The TunnelState CloudWatch metric reports 1 for up and 0 for down."
---

## Phase 1 â€” Triage

MUST:
- Check VPN connection status: `aws ec2 describe-vpn-connections --vpn-connection-id <vpn-id> --query 'VpnConnections[].VgwTelemetry'`
- Check both tunnel statuses (Tunnel1 and Tunnel2)
- Verify customer gateway configuration: `aws ec2 describe-customer-gateways --customer-gateway-id <cgw-id>`
- Check CloudWatch TunnelState metric: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id> --start-time <time> --end-time <time> --period 300 --statistics Average`

SHOULD:
- Check VPN connection options for IKE version and encryption settings
- Verify the customer gateway public IP is reachable from the internet
- Check if the VPN is attached to a VGW or TGW

MAY:
- Download the VPN configuration file for the customer gateway device
- Check customer gateway device logs for IKE errors
- Verify no NAT device is between the customer gateway and the internet

## Phase 2 â€” Remediate

MUST:
- Verify pre-shared key matches on both sides
- Ensure customer gateway public IP is correct and reachable
- Configure both tunnels on the customer gateway (not just one)
- Verify IKE version and encryption parameters match

SHOULD:
- Enable Dead Peer Detection (DPD) on the customer gateway
- Configure a keep-alive mechanism to prevent idle timeout
- Check that UDP 500, UDP 4500, and IP protocol 50 (ESP) are allowed

MAY:
- Reboot the customer gateway device if configuration is correct but tunnel won't establish
- Modify VPN tunnel options if IKE parameters need adjustment

## Common Issues

- symptoms: "Both tunnels show DOWN in VgwTelemetry"
  diagnosis: "IKE negotiation failing. Pre-shared key mismatch, wrong IKE version, or firewall blocking."
  resolution: "Verify PSK, IKE version, and ensure UDP 500/4500 and ESP are allowed through firewalls."

- symptoms: "One tunnel UP, one tunnel DOWN"
  diagnosis: "Only one tunnel configured on customer gateway, or second tunnel has config mismatch."
  resolution: "Configure both tunnels on the customer gateway with correct parameters for each."

- symptoms: "Tunnel was UP but went DOWN during maintenance"
  diagnosis: "Only one tunnel was configured. AWS maintenance took down the active tunnel."
  resolution: "Configure both tunnels. AWS performs maintenance on one tunnel at a time."

## Output Format

```yaml
root_cause: "tunnel_down â€” <specific_cause>"
evidence:
  - type: vgw_telemetry
    content: "<tunnel status and last status change>"
  - type: customer_gateway
    content: "<CGW configuration>"
severity: CRITICAL
mitigation:
  immediate: "Fix tunnel configuration and restore connectivity"
  long_term: "Configure both tunnels and implement tunnel monitoring"
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
