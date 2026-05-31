---
title: "G1 — VPN Tunnel Down"
description: "Diagnose Site-to-Site VPN tunnel connectivity failures"
status: active
severity: CRITICAL
triggers:
  - "VPN tunnel DOWN"
  - "IPsec negotiation failed"
  - "On-premises connectivity lost"
owner: devops-agent
objective: "Restore VPN tunnel connectivity"
context: "AWS Site-to-Site VPN has two tunnels for redundancy. Each tunnel has IKE (Phase 1) and IPsec (Phase 2) negotiation. Common failures: mismatched pre-shared keys, mismatched encryption algorithms, incorrect peer IP, NAT-T issues."
---

## Phase 1 — Triage

MUST:
- Check VPN connection state: `aws ec2 describe-vpn-connections --vpn-connection-ids <id>`
- Check tunnel status in the output: Tunnel1/Tunnel2 → Status (UP/DOWN)
- Check tunnel status details for error messages
- Verify customer gateway IP matches the on-premises device

SHOULD:
- Check if both tunnels are down or just one
- Verify IKE version matches (IKEv1 vs IKEv2)
- Check if pre-shared key was recently rotated
- Verify encryption/integrity/DH group settings match on both sides

MAY:
- Check CloudWatch metrics: TunnelState, TunnelDataIn, TunnelDataOut
- Review VPN connection configuration download for correct parameters
- Check if on-premises firewall allows UDP 500, UDP 4500 (NAT-T), and IP protocol 50 (ESP)

## Common Issues

- symptoms: "Both tunnels DOWN after customer gateway device replacement"
  diagnosis: "New device has different public IP or different IKE configuration."
  resolution: "Update customer gateway with new IP, or reconfigure device to match AWS VPN settings."

- symptoms: "Tunnel flaps (UP/DOWN repeatedly)"
  diagnosis: "DPD (Dead Peer Detection) timeout mismatch or unstable internet connection."
  resolution: "Adjust DPD settings, check internet stability, verify keepalive settings."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN tunnel status check"
  - "describe-customer-gateways: GREEN — read-only customer gateway inspection"
  - "get-metric-statistics (TunnelState): GREEN — read-only CloudWatch metric query"
  - "modify-vpn-tunnel-options (fix IKE/IPsec settings): YELLOW — changes tunnel config, recoverable by reverting options"
  - "modify-vpn-connection (change target gateway): RED — changes VPN target, causes tunnel downtime"
  - "delete-vpn-connection: RED — deletes VPN, breaks all on-premises connectivity"
```

## Escalation Conditions

- "Fix requires modifying VPN tunnel options in production"
- "Both tunnels are DOWN — all on-premises connectivity lost"
- "Fix requires coordination with on-premises network team"
- "Fix involves changing IKE/IPsec parameters — requires matching changes on both sides"

## Data Sensitivity

- HIGH: VPN connection configuration (exposes pre-shared keys, peer IPs, encryption settings)
- HIGH: customer gateway configuration (exposes on-premises network details)
- MEDIUM: CloudWatch tunnel metrics, IKE/IPsec negotiation parameters

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER expose VPN pre-shared keys in logs or output"
- "NEVER delete a VPN connection without confirming backup connectivity exists"

## Phase 3 — Rollback

- If VPN tunnel options were modified: revert with `aws ec2 modify-vpn-tunnel-options --vpn-connection-id <id> --vpn-tunnel-outside-ip-address <ip> --tunnel-options <original-options>`
- If customer gateway was updated: revert on-premises device to previous IKE/IPsec configuration
- If VPN connection target was changed: change it back with `aws ec2 modify-vpn-connection --vpn-connection-id <id> --vpn-gateway-id <original-vgw-id>`
- Document VPN configuration, tunnel options, and customer gateway settings before changes

## Output Format

```yaml
root_cause: "vpn_tunnel_down — <IKE/IPsec/config detail>"
evidence:
  - type: vpn_connection
    content: "<tunnel status and error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix IKE/IPsec configuration mismatch"
  long_term: "Monitor tunnel state with CloudWatch alarms, use both tunnels for redundancy"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
