---
title: "E1 — Tunnel Health Checks"
description: "Diagnose VPN tunnel health check failures and monitoring issues"
status: active
severity: HIGH
triggers: ["tunnel health check failed", "tunnel flapping", "tunnel intermittent", "VPN tunnel unstable"]
owner: devops-agent
objective: "Resolve VPN tunnel health check failures and stability issues"
context: "VPN tunnel health is monitored via VgwTelemetry, CloudWatch TunnelState metric, and DPD. Tunnel flapping triggers BGP dampening and affects route stability."
---

## Phase 1 — Triage

MUST:
- Check tunnel telemetry: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry[*].{OutsideIp:OutsideIpAddress,Status:Status,StatusMessage:StatusMessage,AcceptedRouteCount:AcceptedRouteCount,LastStatusChange:LastStatusChange}'`
- Check CloudWatch TunnelState metric history: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Verify both tunnels are configured on the customer gateway

SHOULD:
- Check VPN logs for DPD and IKE events: `aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --filter-pattern "DPD" --limit 30`
- Review tunnel state change frequency to identify flapping patterns
- Verify IKE rekey timers are compatible between AWS and customer gateway

MAY:
- Check network path between customer gateway and AWS tunnel endpoints
- Review customer gateway system logs for hardware or software issues

## Phase 2 — Remediate

MUST:
- If tunnel DOWN: verify IKE parameters match between AWS and customer gateway
- If tunnel flapping: identify the root cause (DPD timeout, IKE rekey failure, network instability)
- Ensure both tunnels are configured for redundancy

SHOULD:
- Align DPD timeout settings between AWS and customer gateway
- Verify IKE Phase 1 and Phase 2 lifetime values are compatible
- Check for MTU issues causing fragmentation

MAY:
- Implement BFD for faster failure detection (if supported by customer gateway)
- Set up CloudWatch alarms for tunnel state changes

## Common Issues

- symptoms: "Tunnel flaps every 30-60 minutes"
  diagnosis: "IKE Phase 2 (IPsec SA) rekey failure. Lifetime mismatch between AWS and customer gateway."
  resolution: "Align IKE Phase 2 lifetime on the customer gateway with AWS settings. AWS default is 1 hour."

- symptoms: "Tunnel shows DOWN but customer gateway shows UP"
  diagnosis: "DPD timeout on AWS side. Customer gateway not responding to DPD probes."
  resolution: "Enable DPD responses on the customer gateway. Verify firewall allows DPD packets (UDP port 500/4500)."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections (telemetry): GREEN — read-only tunnel status"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log query"
  - "modify-vpn-tunnel-options: RED — changes tunnel parameters, causes tunnel restart"
```

## Escalation Conditions

- Both tunnels flapping simultaneously causing complete VPN outage
- Tunnel instability persists after aligning IKE and DPD parameters
- Customer gateway hardware failure suspected
- Tunnel health issues affect production traffic with no alternative connectivity

## Data Sensitivity

- `aws ec2 describe-vpn-connections (telemetry)`: MEDIUM — reveals tunnel endpoints and status
- `aws cloudwatch get-metric-statistics`: LOW — reveals tunnel state history
- `aws logs filter-log-events`: MEDIUM — reveals IKE/DPD negotiation details

## Prohibited Actions

- NEVER modify tunnel options on both tunnels simultaneously — always keep one tunnel stable
- NEVER restart both tunnels at the same time for troubleshooting
- NEVER change IKE parameters in production without a maintenance window
- NEVER ignore tunnel flapping — it causes BGP dampening and route instability

## Phase 3 — Rollback

1. If tunnel options were modified, revert to previous settings one tunnel at a time
2. Verify the reverted tunnel establishes successfully before modifying the second
3. Check BGP route propagation is restored after tunnel stabilization
4. Verify CloudWatch TunnelState shows consistent UP status
5. Document the root cause and resolution for future reference

## Output Format

```yaml
root_cause: "tunnel_health — <specific_cause>"
evidence:
  - type: tunnel_state
    content: "<telemetry and metric details>"
severity: HIGH
mitigation:
  immediate: "Stabilize tunnel by fixing IKE/DPD parameters"
  long_term: "Implement tunnel monitoring and proactive alerting"
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
