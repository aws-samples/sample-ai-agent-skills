---
title: "E2 — DPD Timeout Issues"
description: "Diagnose Dead Peer Detection (DPD) timeout and tunnel keepalive issues"
status: active
severity: HIGH
triggers: ["DPD timeout", "dead peer detection", "tunnel keepalive failed", "DPD causing tunnel down", "peer unreachable"]
owner: devops-agent
objective: "Resolve DPD timeout mismatches and tunnel keepalive issues"
context: "DPD detects when a VPN peer becomes unreachable. Mismatched DPD settings between AWS and the customer gateway cause tunnel flapping. DPD action (restart, clear, none) varies by vendor."
---

## Phase 1 — Triage

MUST:
- Check VPN tunnel options for DPD settings: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Options.TunnelOptions[*].DpdTimeoutSeconds'`
- Check tunnel telemetry for DPD-related status messages: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`
- Check VPN logs for DPD events: `aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --filter-pattern "DPD" --limit 30`

SHOULD:
- Verify customer gateway DPD settings (timeout, action, interval)
- Check if DPD action is set to restart, clear, or none on the customer gateway
- Review tunnel state change frequency correlating with DPD timeouts

MAY:
- Check network latency between customer gateway and AWS tunnel endpoints
- Verify firewall rules allow IKE keepalive packets (UDP 500/4500)

## Phase 2 — Remediate

MUST:
- Align DPD timeout between AWS and customer gateway (AWS default: 30 seconds)
- Set DPD action to "restart" on the customer gateway for automatic recovery
- Verify firewall allows UDP ports 500 and 4500 bidirectionally

SHOULD:
- Modify AWS DPD timeout if needed: `aws ec2 modify-vpn-tunnel-options --vpn-connection-id <vpn-id> --vpn-tunnel-outside-ip-address <ip> --tunnel-options DpdTimeoutSeconds=30,DpdTimeoutAction=clear`
- Test DPD by temporarily blocking traffic and verifying recovery

MAY:
- Increase DPD timeout for high-latency links
- Implement monitoring for DPD-related tunnel drops

## Common Issues

- symptoms: "Tunnel drops every few minutes with DPD timeout"
  diagnosis: "Customer gateway DPD timeout shorter than AWS. CGW declares peer dead before AWS responds."
  resolution: "Increase customer gateway DPD timeout to match or exceed AWS DPD timeout (default 30s)."

- symptoms: "Tunnel goes DOWN and does not recover automatically"
  diagnosis: "DPD action set to 'clear' or 'none' on customer gateway instead of 'restart'."
  resolution: "Set DPD action to 'restart' on the customer gateway to automatically re-establish the tunnel."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections (DPD): GREEN — read-only DPD configuration"
  - "logs filter-log-events: GREEN — read-only log query"
  - "modify-vpn-tunnel-options (DPD): RED — changes DPD settings, causes tunnel restart"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
```

## Escalation Conditions

- DPD timeouts persist after aligning settings on both sides
- Customer gateway vendor does not support configurable DPD settings
- DPD-related tunnel drops affect production traffic with no redundancy
- Network path issues (high latency, packet loss) causing DPD failures

## Data Sensitivity

- `aws ec2 describe-vpn-connections (DPD)`: MEDIUM — reveals DPD configuration
- `aws logs filter-log-events (DPD)`: MEDIUM — reveals DPD event details
- Customer gateway DPD configuration: MEDIUM — reveals keepalive settings

## Prohibited Actions

- NEVER set DPD timeout to 0 (disables DPD) — this prevents dead peer detection
- NEVER modify DPD settings on both tunnels simultaneously
- NEVER change DPD action to 'none' in production — tunnels will not recover from failures
- NEVER assume DPD issues are always on the AWS side — check the customer gateway first

## Phase 3 — Rollback

1. If DPD settings were modified on AWS, revert: `aws ec2 modify-vpn-tunnel-options --vpn-connection-id <vpn-id> --vpn-tunnel-outside-ip-address <ip> --tunnel-options DpdTimeoutSeconds=<original>`
2. Wait for the tunnel to re-establish after the modification
3. Verify the second tunnel remained stable during the change
4. Confirm DPD is functioning correctly by monitoring tunnel state
5. Document DPD configuration for both sides

## Output Format

```yaml
root_cause: "dpd_timeout — <specific_cause>"
evidence:
  - type: dpd_config
    content: "<DPD settings and event details>"
severity: HIGH
mitigation:
  immediate: "Align DPD settings between AWS and customer gateway"
  long_term: "Implement DPD monitoring and vendor-specific configuration guides"
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
