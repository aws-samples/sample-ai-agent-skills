---
title: "D1 — BGP Route Propagation"
description: "Diagnose BGP route propagation delays, failures, and dampening issues"
status: active
severity: HIGH
triggers: ["BGP routes missing", "route propagation failed", "BGP session down", "routes not propagating", "BGP dampening"]
owner: devops-agent
objective: "Resolve BGP route propagation issues between on-premises and AWS VPN"
context: "BGP route propagation can take up to 60 seconds. Route flapping triggers dampening (up to 60 minutes suppression). BGP is required for ECMP and recommended for automatic failover."
---

## Phase 1 — Triage

MUST:
- Check VPN tunnel BGP status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry[*].{OutsideIp:OutsideIpAddress,Status:Status,StatusMessage:StatusMessage}'`
- Check TGW route table for propagated routes: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb-id> --filters Name=type,Values=propagated`
- Verify BGP ASN configuration: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Options.{LocalBgpAsn:LocalIpv4NetworkCidr,RemoteBgpAsn:RemoteIpv4NetworkCidr}'`

SHOULD:
- Check for BGP dampening by looking at route age and stability
- Verify the customer gateway is advertising the expected prefixes
- Check if route propagation is enabled on the TGW route table: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <rtb-id>`

MAY:
- Review CloudWatch VPN metrics for tunnel flapping history
- Check customer gateway BGP logs for session establishment details

## Phase 2 — Remediate

MUST:
- If BGP session DOWN: verify IKE tunnel is UP first, then check BGP ASN and peer IP configuration
- If routes not propagating: enable route propagation on the TGW route table
- If dampening suspected: wait for dampening timer to expire (up to 60 minutes)

SHOULD:
- Stabilize the tunnel to prevent further BGP flapping
- Verify the customer gateway is advertising correct prefixes with correct AS path

MAY:
- Adjust BGP timers on the customer gateway for faster detection
- Implement BFD (Bidirectional Forwarding Detection) for faster failover

## Common Issues

- symptoms: "BGP routes appear and disappear intermittently"
  diagnosis: "Tunnel flapping causing BGP dampening on the AWS side."
  resolution: "Stabilize the tunnel (fix DPD, IKE, or network issues). Wait for dampening to clear (up to 60 minutes)."

- symptoms: "BGP session UP but no routes received from on-premises"
  diagnosis: "Customer gateway not advertising prefixes or advertising with wrong AS path."
  resolution: "Verify customer gateway BGP configuration. Ensure prefixes are being advertised to the AWS peer."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN and BGP details"
  - "search-transit-gateway-routes: GREEN — read-only route search"
  - "get-transit-gateway-route-table-propagations: GREEN — read-only propagation check"
  - "enable-transit-gateway-route-table-propagation: YELLOW — enables route propagation"
  - "create-transit-gateway-route: YELLOW — adds static route to TGW"
```

## Escalation Conditions

- BGP session cannot be established despite correct configuration on both sides
- BGP dampening persists for extended periods affecting production traffic
- Route propagation causes unexpected routes in the TGW route table
- BGP route leaking between VPN connections affecting network segmentation

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN and BGP configuration
- `aws ec2 search-transit-gateway-routes`: MEDIUM — reveals routing topology
- `aws ec2 get-transit-gateway-route-table-propagations`: LOW — reveals propagation settings

## Prohibited Actions

- NEVER add static routes as a permanent fix for BGP propagation issues
- NEVER modify BGP ASN on a production VPN connection without a maintenance window
- NEVER disable route propagation on a TGW route table without understanding the impact
- NEVER assume BGP dampening is a bug — it is a protection mechanism against route instability

## Phase 3 — Rollback

1. If route propagation was enabled/disabled in error, revert the setting
2. If static routes were added as a workaround, remove them once BGP is restored
3. Verify the TGW route table contains the expected propagated routes
4. Confirm end-to-end connectivity from on-premises to AWS
5. Document BGP configuration changes and notify the network team

## Output Format

```yaml
root_cause: "bgp_route_propagation — <specific_cause>"
evidence:
  - type: bgp_state
    content: "<BGP session and route details>"
severity: HIGH
mitigation:
  immediate: "Fix BGP configuration or wait for dampening to clear"
  long_term: "Implement BGP monitoring and tunnel stability improvements"
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
