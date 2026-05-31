---
title: "D1 — Gateway Connectivity"
description: "Diagnose gateway internet connectivity and AWS communication issues"
status: active
severity: HIGH
triggers: ["gateway offline", "gateway disconnected", "no internet", "gateway LED red", "connection lost"]
owner: devops-agent
objective: "Restore gateway internet connectivity and AWS communication"
context: "Gateways require continuous internet connectivity to transmit sensor data to AWS. Connectivity loss causes data gaps for all sensors connected to the gateway."
---

## Phase 1 — Triage

MUST:
- Check gateway LED status (solid green = connected, blinking = connecting, red = error)
- Verify WiFi network is operational from the gateway location
- Check if all sensors on this gateway have data gaps: `aws cloudwatch get-metric-statistics --namespace AWS/Monitron --metric-name SensorDataPoints --dimensions Name=ProjectId,Value=<project-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Test internet connectivity from the same WiFi network
- Check if WiFi password was recently changed
- Verify no firewall changes blocking outbound HTTPS

MAY:
- Check CloudTrail for gateway disconnect events
- Review network infrastructure logs for WiFi access point issues

## Phase 2 — Remediate

MUST:
- Restore WiFi connectivity — verify SSID, password, and 2.4 GHz band
- Ensure outbound HTTPS (port 443) to AWS endpoints is not blocked
- Power cycle the gateway if LED shows error state (unplug 30 seconds, replug)

SHOULD:
- Verify DNS resolution works from the gateway network
- Check WiFi signal strength at the gateway location
- Confirm no MAC address filtering blocks the gateway

MAY:
- Set up a dedicated SSID for Monitron gateways
- Configure network monitoring to alert on gateway disconnections

## Common Issues

- symptoms: "Gateway LED is solid red"
  diagnosis: "WiFi connection failed — network unreachable or credentials wrong."
  resolution: "Verify WiFi is operational. Re-enter credentials through the mobile app if password changed."

- symptoms: "Gateway LED blinks continuously without going solid green"
  diagnosis: "WiFi connects but internet access is blocked (firewall, proxy, or DNS)."
  resolution: "Check firewall rules for outbound HTTPS. Verify DNS resolution. Ensure no proxy authentication required."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "Gateway power cycle: YELLOW — temporarily disrupts all connected sensors"
  - "WiFi reconfiguration via app: YELLOW — may temporarily disconnect gateway"
  - "Gateway factory reset: RED — removes all sensor pairings"
```

## Escalation Conditions

- Gateway remains offline after WiFi verification and power cycle
- Network infrastructure changes require IT team involvement
- Multiple gateways go offline simultaneously (network-wide issue)
- Gateway hardware appears defective (no LED activity)

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metrics
- WiFi network details: HIGH — network credentials and configuration
- Gateway MAC address: LOW — device identifier
- Network infrastructure logs: MEDIUM — reveals network topology

## Prohibited Actions

- NEVER factory reset a gateway as a first troubleshooting step
- NEVER connect gateways to unsecured or public WiFi networks
- NEVER share WiFi credentials in troubleshooting logs
- NEVER modify network firewall rules without IT team approval

## Phase 3 — Rollback

1. If WiFi was reconfigured incorrectly, re-enter correct credentials through the app
2. If gateway was power cycled, verify all sensors reconnect (check data flow)
3. If firewall rules were changed, revert if gateway connectivity is not restored
4. Document the connectivity issue timeline for maintenance records
5. Verify data collection resumes for all sensors after connectivity is restored

## Output Format

```yaml
root_cause: "gateway_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<LED status, WiFi details, CloudWatch metrics>"
severity: HIGH
mitigation:
  immediate: "Restore WiFi connectivity and power cycle"
  long_term: "Implement network monitoring for gateway connectivity"
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
