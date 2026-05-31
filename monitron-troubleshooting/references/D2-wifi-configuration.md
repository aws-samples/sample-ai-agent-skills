---
title: "D2 — WiFi Configuration"
description: "Diagnose WiFi configuration issues specific to Monitron gateway requirements"
status: active
severity: HIGH
triggers: ["WiFi error", "cannot connect WiFi", "wrong band", "WPA error", "enterprise WiFi"]
owner: devops-agent
objective: "Configure WiFi to meet Monitron gateway requirements"
context: "Monitron gateways require 2.4 GHz WiFi with WPA2 security and outbound internet access. Enterprise WiFi, captive portals, and 5 GHz-only networks are not supported."
---

## Phase 1 — Triage

MUST:
- Verify WiFi band is 2.4 GHz (not 5 GHz only)
- Confirm security protocol is WPA2 (not WPA3-only or open)
- Test outbound internet access from the WiFi network

SHOULD:
- Check if WiFi uses enterprise authentication (802.1X) — not supported
- Verify no captive portal is required for network access
- Confirm WiFi SSID is broadcasting (hidden SSIDs require manual config)

MAY:
- Check WiFi signal strength at the gateway location
- Review WiFi access point configuration for band steering settings
- Test with a simple WPA2 personal network to isolate the issue

## Phase 2 — Remediate

MUST:
- Configure a 2.4 GHz WiFi network with WPA2-Personal security
- Ensure outbound HTTPS (port 443) is allowed to AWS endpoints
- Provide WiFi credentials through the Monitron mobile app

SHOULD:
- Disable band steering on the access point or create a 2.4 GHz-only SSID
- Ensure WiFi channel does not conflict with other 2.4 GHz devices
- Position access point within reasonable range of the gateway

MAY:
- Create a dedicated SSID for IoT devices including Monitron gateways
- Configure QoS to prioritize Monitron gateway traffic
- Set up a WiFi site survey for optimal access point placement

## Common Issues

- symptoms: "Gateway cannot see the WiFi network"
  diagnosis: "WiFi is 5 GHz only or SSID is hidden."
  resolution: "Enable 2.4 GHz band on the access point. If SSID is hidden, manually enter SSID in the app."

- symptoms: "Gateway connects to WiFi but cannot reach AWS"
  diagnosis: "Firewall blocks outbound HTTPS or proxy authentication required."
  resolution: "Allow outbound HTTPS to *.amazonaws.com. Ensure no proxy authentication is required for the gateway."

## Safety Ratings

```
safety_ratings:
  - "WiFi network testing: GREEN — passive network observation"
  - "WiFi reconfiguration via app: YELLOW — may temporarily disconnect gateway"
  - "Access point configuration: YELLOW — affects all devices on the network"
  - "Firewall rule changes: YELLOW — affects network security posture"
```

## Escalation Conditions

- Facility only has enterprise WiFi (802.1X) with no option for WPA2-Personal
- Network security policy prohibits creating dedicated IoT SSIDs
- Firewall changes require security team approval
- WiFi signal strength is insufficient at the gateway location

## Data Sensitivity

- WiFi SSID and configuration: MEDIUM — network infrastructure details
- WiFi credentials: HIGH — network access credentials
- Firewall rules: MEDIUM — reveals network security configuration
- Access point placement: LOW — physical infrastructure details

## Prohibited Actions

- NEVER use open (unsecured) WiFi networks for Monitron gateways
- NEVER share WiFi credentials in troubleshooting documentation
- NEVER modify enterprise WiFi configuration without IT team approval
- NEVER disable WiFi security to troubleshoot connectivity

## Phase 3 — Rollback

1. If WiFi configuration was changed on the access point, restore previous settings
2. If a dedicated SSID was created and is not needed, remove it
3. If firewall rules were modified, revert if not needed
4. Verify other devices on the network are not affected by changes
5. Document the WiFi configuration for the facilities team

## Output Format

```yaml
root_cause: "wifi_configuration — <specific_cause>"
evidence:
  - type: wifi_error
    content: "<WiFi band, security protocol, connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Configure compliant WiFi and reconnect gateway"
  long_term: "Document WiFi requirements for facility network planning"
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
