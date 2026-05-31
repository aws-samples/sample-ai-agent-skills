---
title: "C1 — SIM Provisioning"
description: "Diagnose SIM card provisioning failures"
status: active
severity: HIGH
triggers: ["SIM not working", "SIM provisioning failed", "ICCID error", "IMSI error"]
owner: devops-agent
objective: "Resolve SIM card provisioning and activation issues"
context: "SIM cards are AWS-provided and must be activated through the Private 5G service. Third-party SIMs are not supported."
---

## Phase 1 — Triage

MUST:
- List device identifiers: `aws private-networks list-device-identifiers --network-arn <arn>`
- Get SIM details: `aws private-networks get-device-identifier --device-identifier-arn <arn>`
- Verify SIM status — should be ACTIVE

SHOULD:
- Check if SIM was included in the hardware order: `aws private-networks list-orders --network-arn <arn>`
- Verify SIM is associated with the correct network
- Review CloudTrail for provisioning events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ActivateDeviceIdentifier --max-results 10`

MAY:
- Cross-reference ICCID on physical SIM card with the registered identifier

## Phase 2 — Remediate

MUST:
- Activate SIM if in INACTIVE status: `aws private-networks activate-device-identifier --device-identifier-arn <arn>`
- Verify SIM is associated with the correct network ARN
- Ensure the physical SIM card matches the registered ICCID

SHOULD:
- Confirm SIM card is not physically damaged
- Verify SIM is properly seated in the device SIM tray

MAY:
- Request replacement SIM from AWS if card is defective
- Order additional SIMs for new devices

## Safety Ratings

```
safety_ratings:
  - "list-device-identifiers: GREEN — read-only enumeration"
  - "get-device-identifier: GREEN — read-only SIM details"
  - "activate-device-identifier: YELLOW — activates SIM for network access"
  - "deactivate-device-identifier: YELLOW — removes device network access"
```

## Escalation Conditions

- SIM activation fails with internal service errors
- SIM shows ACTIVE but device cannot authenticate to the network
- Multiple SIMs from the same order fail to activate
- SIM ICCID does not match any registered device identifier

## Data Sensitivity

- `aws private-networks list-device-identifiers`: MEDIUM — reveals IMSI/ICCID values
- `aws private-networks get-device-identifier`: MEDIUM — reveals subscriber identity
- SIM card ICCID/IMSI: HIGH — unique subscriber identifiers

## Prohibited Actions

- NEVER share IMSI or ICCID values in public channels or logs
- NEVER activate SIMs on a network without authorization from the network owner
- NEVER attempt to use third-party SIM cards with Private 5G
- NEVER discard SIM cards without proper deactivation

## Phase 3 — Rollback

1. If a SIM was activated in error, deactivate it: `aws private-networks deactivate-device-identifier --device-identifier-arn <arn>`
2. If SIM was associated with the wrong network, deactivate and re-provision on the correct network
3. Verify no unauthorized devices gained network access during troubleshooting
4. Confirm legitimate devices are still connected after changes
5. Document SIM provisioning changes and update inventory

## Common Issues

- symptoms: "ActivateDeviceIdentifier returns ResourceNotFoundException"
  diagnosis: "SIM ICCID is not registered in the network or was not included in the hardware order."
  resolution: "Verify ICCID matches a registered device identifier. Order additional SIMs if needed."

- symptoms: "SIM shows ACTIVE but device shows 'SIM not provisioned'"
  diagnosis: "Device may need restart to pick up SIM activation, or SIM is not properly inserted."
  resolution: "Restart the device. Re-seat the SIM card. Verify device supports CBRS."

## Output Format

```yaml
root_cause: "sim_provisioning — <specific_cause>"
evidence:
  - type: sim_status
    content: "<status details>"
severity: HIGH
mitigation:
  immediate: "Resolve SIM provisioning issue"
  long_term: "Maintain SIM inventory and activation records"
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
