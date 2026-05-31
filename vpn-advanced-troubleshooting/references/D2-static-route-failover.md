---
title: "D2 — Static Route Failover"
description: "Diagnose static route VPN failover issues and route management"
status: active
severity: HIGH
triggers: ["static route failover not working", "VPN failover failed", "static route VPN down", "backup tunnel not active"]
owner: devops-agent
objective: "Resolve static route VPN failover and route management issues"
context: "Static route VPN connections do not automatically failover like BGP. Route management is manual. For TGW-based VPNs, static routes can be configured on the TGW route table."
---

## Phase 1 — Triage

MUST:
- Check VPN connection routing type: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Options.StaticRoutesOnly'`
- Check static routes on the VPN: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Routes'`
- Check tunnel status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`

SHOULD:
- For TGW VPN: check TGW route table static routes: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb-id> --filters Name=type,Values=static`
- For VGW VPN: check VPC route table routes pointing to the VGW
- Verify both tunnels are configured on the customer gateway

MAY:
- Check if health check-based routing is configured for automatic failover
- Review CloudWatch metrics for tunnel state history

## Phase 2 — Remediate

MUST:
- Ensure static routes are configured for both tunnels
- For VGW VPN: verify VPC route table has routes pointing to the VGW
- For TGW VPN: verify TGW route table has static routes for on-premises prefixes

SHOULD:
- Configure both tunnels on the customer gateway for redundancy
- Consider migrating to BGP for automatic failover

MAY:
- Implement health check-based routing using Route 53 or custom scripts
- Set up CloudWatch alarms for tunnel state to trigger manual failover

## Common Issues

- symptoms: "Traffic stops when primary tunnel goes down despite backup tunnel being UP"
  diagnosis: "Static routes point to the primary tunnel only. No automatic failover for static routing."
  resolution: "Configure routes for both tunnels. For VGW VPN, AWS handles failover. For TGW, ensure both tunnels have routes."

- symptoms: "Static routes not appearing in VPC route table"
  diagnosis: "Route propagation not enabled on the VPC route table for the VGW."
  resolution: "Enable route propagation: aws ec2 enable-vgw-route-propagation --route-table-id <rtb-id> --gateway-id <vgw-id>."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN details"
  - "search-transit-gateway-routes: GREEN — read-only route search"
  - "create-vpn-connection-route: YELLOW — adds static route to VPN"
  - "create-transit-gateway-route: YELLOW — adds static route to TGW"
  - "enable-vgw-route-propagation: YELLOW — enables route propagation in VPC"
```

## Escalation Conditions

- Static route failover not working despite both tunnels being configured
- Manual failover required during off-hours with no on-call network engineer
- Static route VPN cannot meet RTO requirements for the application
- Migration to BGP required but customer gateway does not support BGP

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration and routes
- `aws ec2 search-transit-gateway-routes`: MEDIUM — reveals routing topology
- VPC route table details: MEDIUM — reveals network routing architecture

## Prohibited Actions

- NEVER remove static routes from a production VPN without confirming backup routes exist
- NEVER assume static route VPNs have automatic failover — they require manual or health-check-based failover
- NEVER modify VPC route tables in production without a rollback plan
- NEVER disable VGW route propagation without understanding the impact on all VPN connections

## Phase 3 — Rollback

1. If static routes were removed, re-add them: `aws ec2 create-vpn-connection-route --vpn-connection-id <vpn-id> --destination-cidr-block <cidr>`
2. If VPC route table was modified, revert routes to previous state
3. If TGW routes were changed, restore previous static routes
4. Verify end-to-end connectivity through both tunnels
5. Document the rollback and review failover strategy

## Output Format

```yaml
root_cause: "static_route_failover — <specific_cause>"
evidence:
  - type: route_state
    content: "<static route and tunnel details>"
severity: HIGH
mitigation:
  immediate: "Fix static route configuration"
  long_term: "Migrate to BGP for automatic failover or implement health-check-based routing"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
