---
title: "E1 — Network Configuration"
description: "Diagnose Snow Family device network configuration issues"
status: active
severity: HIGH
triggers: ["network error", "cannot connect to device", "IP configuration", "network unreachable"]
owner: devops-agent
objective: "Resolve device network configuration issues including IP assignment, routing, and interface selection"
context: "Snow devices support RJ45, SFP+, and QSFP28 interfaces. Network configuration is done locally via Snowball Edge client or LCD display."
---

## Phase 1 — Triage

MUST:
- Check device IP from LCD display or local network scan
- Test connectivity: `ping <device-ip>` and `curl -k https://<device-ip>:8243`
- Verify physical cable connection and link status

SHOULD:
- Check which network interface is active (RJ45, SFP+, QSFP28)
- Verify IP address, subnet mask, and gateway configuration

MAY:
- Check DHCP server if using dynamic IP assignment
- Review network switch port configuration

## Phase 2 — Remediate

MUST:
- Configure IP address via LCD display or Snowball Edge client
- Ensure device is on the same subnet as the transfer host (or routing is configured)
- Verify physical cable is properly connected and link light is active

SHOULD:
- Use static IP for production deployments
- Configure the fastest available network interface for data transfer

MAY:
- Create virtual network interfaces for service isolation: `snowballEdge create-virtual-network-interface --physical-network-interface-id <id> --ip-address-assignment DHCP`

## Safety Ratings

```
safety_ratings:
  - "ping: GREEN — read-only connectivity test"
  - "curl (device endpoint): GREEN — read-only endpoint test"
  - "create-virtual-network-interface: YELLOW — creates network interface on device"
  - "LCD network configuration: YELLOW — modifies device network settings"
```

## Escalation Conditions

- Device network interface is physically damaged
- Network configuration requires changes to corporate network infrastructure
- Device cannot obtain DHCP address and static IP is not available
- Multiple network interfaces need configuration for different services

## Data Sensitivity

- Network configuration: LOW — reveals IP addressing only
- `snowballEdge describe-device`: MEDIUM — reveals all network interface details
- Network traffic: HIGH — data in transit between host and device

## Prohibited Actions

- NEVER expose Snow device management port (8243) to the internet
- NEVER change network configuration during active data transfers
- NEVER use public IP addresses for Snow device interfaces
- NEVER disable all network interfaces simultaneously

## Phase 3 — Rollback

1. If network configuration was changed, restore previous IP settings via LCD
2. If virtual network interfaces were created incorrectly, delete them
3. Verify device is accessible after network changes
4. Restore original network switch configuration if modified
5. Document the correct network configuration for the deployment

## Common Issues

- symptoms: "Cannot reach device IP address after power on"
  diagnosis: "Device has no IP configured or is on a different subnet."
  resolution: "Check LCD display for current IP. Configure IP via LCD or connect directly with a crossover cable."

## Output Format

```yaml
root_cause: "network_configuration — <specific_cause>"
evidence:
  - type: network_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix network config"
  long_term: "Document network setup"
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
