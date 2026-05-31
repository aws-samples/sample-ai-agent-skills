---
title: "G2 — BGP Issues"
description: "Diagnose BGP peering and route propagation issues with VPN and Direct Connect"
status: active
severity: HIGH
triggers:
  - "BGP session down"
  - "Routes not propagating"
  - "On-premises routes missing"
owner: devops-agent
objective: "Restore BGP peering and route propagation"
context: "Dynamic VPN and Direct Connect use BGP for route exchange. AWS uses ASN 64512 by default (configurable). BGP sessions require matching ASN configuration, correct peer IPs, and allowed TCP 179."
---

## Phase 1 — Triage

MUST:
- Check VPN tunnel BGP status in describe-vpn-connections output
- Verify customer gateway ASN matches configuration
- Check if routes are being advertised from on-premises
- Verify VGW/TGW route propagation is enabled

SHOULD:
- Check if BGP hold timer is expiring (default 30 seconds)
- Verify on-premises device is advertising the correct prefixes
- Check if route limits are exceeded (100 routes per VPN connection)

MAY:
- Check Direct Connect virtual interface BGP status
- Verify BGP communities for route preference
- Check if AS-path prepending is affecting route selection

## Common Issues

- symptoms: "BGP session established but no routes received"
  diagnosis: "On-premises device not advertising routes, or route propagation not enabled on VGW/TGW."
  resolution: "Configure route advertisements on customer device and enable propagation."

- symptoms: "BGP session keeps dropping"
  diagnosis: "Hold timer expiring due to packet loss or misconfigured keepalive."
  resolution: "Check network stability, adjust BGP timers, verify TCP 179 is allowed."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections (BGP status): GREEN — read-only VPN/BGP status check"
  - "describe-virtual-interfaces (DX BGP): GREEN — read-only DX VIF BGP check"
  - "get-metric-statistics (TunnelState): GREEN — read-only CloudWatch metric query"
  - "enable-vgw-route-propagation: YELLOW — enables route propagation, recoverable by disabling"
  - "disable-vgw-route-propagation: YELLOW — disables propagation, recoverable by re-enabling"
  - "create-transit-gateway-route (add static route as BGP workaround): YELLOW — adds route, recoverable by deleting"
```

## Escalation Conditions

- "Fix requires coordination with on-premises network team for BGP changes"
- "BGP session down — on-premises routes not propagating to VPC"
- "Fix requires modifying Transit Gateway route tables"
- "Route limit exceeded (100 routes per VPN) — requires route summarization"

## Data Sensitivity

- HIGH: BGP configuration (exposes ASN, peer IPs, advertised prefixes)
- HIGH: route propagation settings (expose hybrid network architecture)
- MEDIUM: CloudWatch metrics, BGP timer settings, AS-path information

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER disable route propagation without confirming static routes exist as backup"

## Phase 3 — Rollback

- If route propagation was enabled: disable it with `aws ec2 disable-vgw-route-propagation --gateway-id <vgw-id> --route-table-id <rtb-id>`
- If route propagation was disabled: re-enable it with `aws ec2 enable-vgw-route-propagation --gateway-id <vgw-id> --route-table-id <rtb-id>`
- If a static TGW route was added: delete it with `aws ec2 delete-transit-gateway-route --transit-gateway-route-table-id <tgw-rtb-id> --destination-cidr-block <cidr>`
- Document BGP status, propagated routes, and route table state before changes

## Output Format

```yaml
root_cause: "bgp_issues — <detail>"
evidence:
  - type: vpn_connection
    content: "<BGP status and route information>"
severity: HIGH
mitigation:
  immediate: "Fix BGP configuration or network connectivity"
  long_term: "Monitor BGP session state, implement BFD for faster failover"
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
