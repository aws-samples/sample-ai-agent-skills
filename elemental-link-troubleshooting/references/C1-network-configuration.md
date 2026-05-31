---
title: "C1 — Network Configuration"
description: "Diagnose Elemental Link network configuration issues"
status: active
severity: HIGH
triggers: ["network error", "firewall blocking", "bandwidth insufficient", "DNS failure", "proxy issue"]
owner: devops-agent
objective: "Resolve network configuration issues affecting Elemental Link connectivity and streaming"
context: "Elemental Link requires outbound HTTPS (443) and sufficient upload bandwidth. Corporate networks with firewalls, proxies, and NAT can interfere with device communication."
---

## Phase 1 — Triage

MUST:
- Check device connection state: `aws medialive describe-input-device --input-device-id <device-id> --query 'ConnectionState'`
- Verify network throughput: `aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=InputDeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify outbound HTTPS connectivity from the device network to AWS endpoints

SHOULD:
- Check for packet loss and latency on the upload path
- Verify DNS resolution for MediaLive endpoints from the device subnet
- Review firewall and proxy logs for blocked connections from the device IP

MAY:
- Test connectivity to AWS IP ranges from a host on the same subnet
- Check if the device has a valid IP address (DHCP lease or static configuration)

## Phase 2 — Remediate

MUST:
- Allow outbound HTTPS (port 443) to AWS IP ranges for the MediaLive service
- Ensure DNS resolution is available on the device network
- Verify the device has a valid IP address with internet routing

SHOULD:
- Bypass proxy servers for AWS endpoint traffic
- Ensure upload bandwidth is at least 2x the target encoding bitrate
- Configure network monitoring for the device subnet

MAY:
- Implement dedicated VLAN for media streaming devices
- Configure QoS policies to prioritize Elemental Link traffic
- Set up redundant internet paths for critical streaming

## Safety Ratings

```
safety_ratings:
  - "describe-input-device: GREEN — read-only device status check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics retrieval"
  - "firewall rule changes: RED — affects network security posture"
  - "proxy bypass configuration: YELLOW — modifies traffic routing"
```

## Escalation Conditions

- Network team unable to identify blocked traffic in firewall logs
- Bandwidth tests show sufficient capacity but device still reports poor connectivity
- DNS resolution works for other services but fails for AWS endpoints
- Multiple devices on different networks experience simultaneous connectivity issues

## Data Sensitivity

- `aws medialive describe-input-device`: LOW — returns device connection metadata
- `aws cloudwatch get-metric-statistics`: LOW — returns network throughput metrics
- Firewall and proxy logs: HIGH — may reveal internal network architecture and security policies
- Device IP address: MEDIUM — reveals internal network addressing

## Prohibited Actions

- NEVER open inbound ports to the Elemental Link device — it uses outbound connections only
- NEVER suggest disabling the firewall entirely to test connectivity
- NEVER expose the device directly to the internet without NAT or firewall protection
- NEVER modify network infrastructure during live production without a maintenance window

## Phase 3 — Rollback

1. Revert firewall rule changes if they did not resolve the issue or introduced security gaps
2. Remove proxy bypass rules if they are no longer needed
3. Revert QoS configuration if it caused issues for other network traffic
4. Restore original VLAN configuration if the dedicated VLAN caused isolation issues
5. Document all network changes made and their outcomes for the network team

## Common Issues

- symptoms: "Device connects intermittently, drops every few minutes"
  diagnosis: "Firewall session timeout too short for persistent HTTPS connections."
  resolution: "Increase firewall session timeout for connections to AWS IP ranges to at least 3600 seconds."

## Output Format

```yaml
root_cause: "network_configuration — <specific_cause>"
evidence:
  - type: network_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix network configuration"
  long_term: "Implement dedicated media network with monitoring"
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
