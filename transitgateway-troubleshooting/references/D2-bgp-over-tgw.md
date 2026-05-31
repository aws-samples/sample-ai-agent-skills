---
title: "D2 — BGP Over TGW Issues"
description: "Diagnose BGP session and route exchange problems over Transit Gateway"
status: active
severity: HIGH
triggers:
  - "BGP session down"
  - "BGP routes not learned"
  - "BGP over TGW not working"
  - "dynamic routing TGW"
owner: devops-agent
objective: "Identify and resolve BGP routing issues over TGW VPN or Connect attachments"
context: "BGP is used for dynamic route exchange over VPN and Connect attachments. BGP sessions require matching ASN, peer IPs, and proper authentication. Routes learned via BGP are propagated to TGW route tables when propagation is enabled."
---

## Phase 1 — Triage

MUST:
- Check VPN tunnel BGP status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry[*].{Tunnel:OutsideIpAddress,Status:Status,StatusMessage:StatusMessage}'`
- Verify BGP routes in TGW route table: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=type,Values=propagated`
- Check the customer gateway ASN: `aws ec2 describe-customer-gateways --customer-gateway-ids <cgw-id> --query 'CustomerGateways[*].BgpAsn'`
- Verify TGW ASN: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.AmazonSideAsn'`

SHOULD:
- Verify route propagation is enabled from the VPN attachment: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <tgw-rtb-id>`
- Check if the customer device is advertising the expected prefixes
- Verify BGP authentication (pre-shared key) matches

MAY:
- Check CloudWatch VPN metrics for BGP route count
- Review VPN connection configuration for BGP parameters

## Phase 2 — Remediate

MUST:
- Ensure BGP ASNs are correctly configured on both sides (must be different)
- Verify BGP peer IP addresses match the tunnel inside addresses
- Enable route propagation from the VPN attachment to the TGW route table

SHOULD:
- Configure BGP on both tunnels for redundancy
- Verify the customer device is advertising the correct prefixes
- Check for BGP route limits (100 routes per VPN connection)

MAY:
- Configure BGP route filtering on the customer device
- Set up BGP monitoring and alerting

## Common Issues

- symptoms: "BGP session established but no routes learned"
  diagnosis: "Customer device not advertising prefixes, or propagation not enabled."
  resolution: "Verify customer device BGP config and enable propagation on TGW route table."

- symptoms: "BGP session flapping"
  diagnosis: "Network instability, MTU issues, or hold timer mismatch."
  resolution: "Check network stability, ensure MTU 1500 for VPN, verify BGP timers."

- symptoms: "Too many BGP routes causing issues"
  diagnosis: "Exceeding the 100-route limit per VPN connection."
  resolution: "Summarize routes on the customer device. Use fewer, broader prefixes."

## Output Format

```yaml
root_cause: "bgp_over_tgw_issue — <specific_cause>"
evidence:
  - type: bgp_status
    content: "<BGP session status per tunnel>"
  - type: learned_routes
    content: "<routes learned via BGP>"
  - type: asn_config
    content: "<ASN configuration on both sides>"
severity: HIGH
mitigation:
  immediate: "Fix BGP configuration or enable propagation"
  long_term: "Monitor BGP session health with CloudWatch alarms"
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
