---
title: "E2 — Optics Connectivity"
description: "Diagnose Snow Family optical network interface and SFP/QSFP connectivity issues"
status: active
severity: MEDIUM
triggers: ["optics error", "SFP not detected", "QSFP error", "no link light", "fiber connectivity"]
owner: devops-agent
objective: "Resolve optical network interface issues including SFP+, QSFP28 transceivers, and fiber connectivity"
context: "Snow devices support SFP+ (10 Gbps) and QSFP28 (25/40/50/100 Gbps) optical interfaces. Transceiver compatibility and fiber type matter."
---

## Phase 1 — Triage

MUST:
- Check physical link status (link light on device and switch)
- Verify transceiver is properly seated in the port
- Confirm fiber cable is connected to correct ports (TX/RX)

SHOULD:
- Verify transceiver compatibility with the Snow device model
- Check switch port configuration matches the transceiver speed

MAY:
- Test with a known-good transceiver and fiber cable
- Check fiber cable for damage or excessive bend radius

## Phase 2 — Remediate

MUST:
- Use compatible SFP+ or QSFP28 transceivers for the device model
- Ensure fiber type matches transceiver type (single-mode vs multi-mode)
- Verify TX/RX fiber connections are not swapped

SHOULD:
- Clean fiber connectors before insertion
- Configure switch port to match transceiver speed and duplex

MAY:
- Use fiber loopback test to verify transceiver functionality
- Upgrade to higher-speed optics for better transfer performance

## Safety Ratings

```
safety_ratings:
  - "Physical inspection: GREEN — visual check only"
  - "Transceiver replacement: YELLOW — requires physical device access"
  - "Switch port configuration: YELLOW — modifies network infrastructure"
  - "Network interface configuration: YELLOW — modifies device networking"
```

## Escalation Conditions

- Transceiver is not recognized by the device despite being compatible
- Fiber infrastructure requires physical plant changes
- Switch port does not support the required speed or protocol
- Multiple optical interfaces fail simultaneously

## Data Sensitivity

- Physical device inspection: LOW — no data exposure
- Network switch configuration: MEDIUM — reveals network topology
- Device network interface details: LOW — reveals interface configuration

## Prohibited Actions

- NEVER force a transceiver into a port — it should insert smoothly
- NEVER look directly into fiber connectors (laser safety hazard)
- NEVER use incompatible transceivers — they may damage the port
- NEVER change switch port configuration without network team approval

## Phase 3 — Rollback

1. If a transceiver was replaced, restore the original if the new one doesn't work
2. If switch port configuration was changed, revert to original settings
3. Verify other devices on the same switch are not affected
4. Test connectivity after any physical changes
5. Document the working transceiver model and fiber type

## Common Issues

- symptoms: "No link light on SFP+ port despite cable connected"
  diagnosis: "Transceiver is incompatible or fiber TX/RX are swapped."
  resolution: "Verify transceiver compatibility. Swap TX and RX fiber connections. Try a different transceiver."

## Output Format

```yaml
root_cause: "optics_connectivity — <specific_cause>"
evidence:
  - type: optics_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix optical connectivity"
  long_term: "Stock compatible transceivers"
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
