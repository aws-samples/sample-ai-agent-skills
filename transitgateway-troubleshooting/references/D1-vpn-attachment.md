---
title: "D1 — VPN Attachment Issues"
description: "Diagnose Transit Gateway VPN attachment problems"
status: active
severity: HIGH
triggers:
  - "VPN attachment not working"
  - "VPN tunnel down on TGW"
  - "VPN attachment failed"
  - "site-to-site VPN TGW"
owner: devops-agent
objective: "Identify and resolve VPN attachment issues on TGW"
context: "VPN attachments connect site-to-site VPN connections to the TGW. Each VPN has two tunnels for redundancy. VPN attachments support BGP for dynamic routing or static routes. MTU is limited to 1500 for VPN."
---

## Phase 1 — Triage

MUST:
- Check VPN connection status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].{Id:VpnConnectionId,State:State,TgwId:TransitGatewayId}'`
- Check tunnel status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`
- Verify the VPN attachment on the TGW: `aws ec2 describe-transit-gateway-attachments --filters Name=resource-type,Values=vpn Name=transit-gateway-id,Values=<tgw-id>`
- Check TGW route table for VPN-learned routes: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=state,Values=active`

SHOULD:
- Verify the customer gateway configuration: `aws ec2 describe-customer-gateways --customer-gateway-ids <cgw-id>`
- Check if BGP is configured and sessions are up
- Verify the VPN attachment is associated with the correct TGW route table

MAY:
- Download VPN configuration for the customer device: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].CustomerGatewayConfiguration'`
- Check CloudWatch VPN metrics: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id>`

## Phase 2 — Remediate

MUST:
- Ensure both tunnels are configured on the customer gateway device
- Verify IKE and IPsec parameters match between AWS and customer device
- Associate the VPN attachment with the correct TGW route table

SHOULD:
- Enable route propagation from the VPN attachment to the TGW route table
- Configure BGP for dynamic route learning
- Set up both tunnels for redundancy

MAY:
- Enable VPN acceleration (if supported in the region)
- Configure VPN logging for troubleshooting

## Common Issues

- symptoms: "VPN tunnels show DOWN status"
  diagnosis: "IKE/IPsec negotiation failure or customer gateway misconfiguration."
  resolution: "Verify IKE version, encryption, authentication, and DH group match on both sides."

- symptoms: "VPN connected but no routes learned"
  diagnosis: "BGP not configured or BGP session not established."
  resolution: "Verify BGP ASN, peer IP, and that route propagation is enabled on the TGW route table."

- symptoms: "VPN attached to TGW but traffic not flowing"
  diagnosis: "VPN attachment not associated with TGW route table, or missing return routes."
  resolution: "Associate VPN attachment with route table and enable propagation."

## Output Format

```yaml
root_cause: "vpn_attachment_issue — <specific_cause>"
evidence:
  - type: vpn_status
    content: "<VPN connection and tunnel status>"
  - type: tgw_attachment
    content: "<VPN attachment state>"
  - type: routes
    content: "<routes learned from VPN>"
severity: HIGH
mitigation:
  immediate: "Fix VPN tunnel or routing configuration"
  long_term: "Monitor VPN health with CloudWatch alarms"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Route tables reveal network topology across accounts
- Peering configurations expose cross-region connectivity
- VPN attachment details contain tunnel configurations
- Classification: INTERNAL - network architecture and routing topology

## Prohibited Actions

- NEVER delete transit gateway route tables with active associations
- NEVER remove VPC or VPN attachments without confirming traffic rerouting
- NEVER modify route propagation settings during business hours without approval
- NEVER change peering configurations without coordinating with peer account owners

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest removing route table associations without review"
  - "NEVER suggest disabling route propagation"
  - "NEVER modify production TGW without maintenance window"
