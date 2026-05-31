---
title: "A2 — Gateway Setup Issues"
description: "Diagnose Monitron gateway commissioning and initial setup failures"
status: active
severity: HIGH
triggers: ["gateway setup failed", "cannot commission gateway", "gateway not connecting", "gateway LED error"]
owner: devops-agent
objective: "Resolve gateway setup and commissioning issues"
context: "Gateway setup requires the Monitron mobile app, 2.4 GHz WiFi with WPA2, and physical proximity to the gateway. Gateways communicate with AWS over the internet."
---

## Phase 1 — Triage

MUST:
- Verify the project exists and is accessible: `aws monitron get-project --project-id <project-id>`
- Confirm the user has admin access to the project: `aws monitron list-project-admin-users --project-id <project-id>`
- Check gateway LED status (physical inspection required — refer to LED guide)

SHOULD:
- Verify WiFi network meets requirements: 2.4 GHz, WPA2, internet access
- Confirm the mobile app is updated to the latest version
- Check if the gateway is powered on and LED is blinking

MAY:
- Test WiFi connectivity from the gateway location with a mobile device
- Check CloudTrail for gateway registration events

## Phase 2 — Remediate

MUST:
- Ensure WiFi is 2.4 GHz band with WPA2 security (not 5 GHz, not WPA3-only)
- Verify internet connectivity from the WiFi network to AWS endpoints
- Use the Monitron mobile app for gateway commissioning (CLI cannot do this)

SHOULD:
- Position gateway within range of target sensors (10-20 meters in industrial settings)
- Ensure gateway has stable power supply (not on a switched outlet)

MAY:
- Set up a dedicated WiFi network for Monitron gateways
- Configure firewall rules to allow outbound HTTPS to AWS endpoints

## Common Issues

- symptoms: "Gateway LED shows solid red during setup"
  diagnosis: "WiFi connection failed — wrong band, wrong security, or no internet."
  resolution: "Verify 2.4 GHz WiFi with WPA2. Test internet access from the same network. Re-enter WiFi credentials in the app."

- symptoms: "Mobile app cannot find the gateway during commissioning"
  diagnosis: "Bluetooth not enabled on phone or gateway not in pairing mode."
  resolution: "Enable Bluetooth on the mobile device. Power cycle the gateway to enter pairing mode. Ensure physical proximity."

## Safety Ratings

```
safety_ratings:
  - "monitron get-project: GREEN — read-only project details"
  - "monitron list-project-admin-users: GREEN — read-only user listing"
  - "Gateway power cycle: YELLOW — temporarily disrupts all connected sensors"
  - "Gateway factory reset: RED — removes all sensor pairings, requires re-setup"
```

## Escalation Conditions

- Gateway hardware appears defective (no LED activity after power on)
- WiFi network requires enterprise authentication (802.1X) that gateway does not support
- Multiple gateways fail to commission in the same location
- Gateway firmware is outdated and cannot connect to current AWS endpoints

## Data Sensitivity

- `aws monitron get-project`: LOW — returns project metadata
- `aws monitron list-project-admin-users`: MEDIUM — reveals admin user identities
- WiFi credentials: HIGH — network access credentials
- Gateway physical location: MEDIUM — reveals facility layout

## Prohibited Actions

- NEVER factory reset a gateway without explicit approval — it removes all sensor pairings
- NEVER share WiFi credentials in logs or documentation
- NEVER connect gateways to public or unsecured WiFi networks
- NEVER attempt to open or modify gateway hardware

## Phase 3 — Rollback

1. If gateway was commissioned to wrong project, decommission through the mobile app
2. If WiFi credentials were entered incorrectly, re-run setup through the app
3. If gateway was factory reset, re-commission and re-pair all sensors (14-day learning period restarts)
4. Document the setup issue for the facilities team
5. Verify all previously connected sensors are re-paired after any gateway reset

## Output Format

```yaml
root_cause: "gateway_setup — <specific_cause>"
evidence:
  - type: setup_error
    content: "<LED status, app error, or connectivity details>"
severity: HIGH
mitigation:
  immediate: "Fix WiFi/connectivity and retry setup"
  long_term: "Document WiFi requirements for facility network team"
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
