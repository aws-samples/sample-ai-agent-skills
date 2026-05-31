---
title: "B1 — Radio Unit Activation"
description: "Diagnose radio unit activation failures"
status: active
severity: CRITICAL
triggers: ["radio unit not active", "radio unit offline", "CBRS activation failed"]
owner: devops-agent
objective: "Resolve radio unit activation and registration issues"
context: "Radio units must be physically installed, powered on, and registered with the CBRS Spectrum Access System (SAS) before becoming active."
---

## Phase 1 — Triage

MUST:
- List network resources: `aws private-networks list-network-resources --network-arn <arn> --filters name=type,values=RADIO_UNIT`
- Get radio unit details: `aws private-networks get-network-resource --network-resource-arn <arn>`
- Verify radio unit status — should be PROVISIONED or ACTIVE

SHOULD:
- Check if hardware has been delivered: `aws private-networks list-orders --network-arn <arn>`
- Verify physical installation: power, ethernet, and antenna connections
- Review CloudWatch metrics for the radio unit: `aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name RadioUnitStatus --dimensions Name=NetworkResourceArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Average`

MAY:
- Check CloudTrail for activation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ActivateNetworkSite --max-results 10`

## Phase 2 — Remediate

MUST:
- Verify physical installation: power supply connected, ethernet cable connected, antennas attached
- Confirm radio unit has network connectivity (can reach AWS endpoints)
- Wait for SAS registration to complete (can take up to 30 minutes)

SHOULD:
- Check for RF interference from other CBRS devices in the area
- Verify the radio unit firmware is up to date (managed by AWS)

MAY:
- Power cycle the radio unit if stuck in registration
- Contact AWS Support if SAS registration fails repeatedly

## Safety Ratings

```
safety_ratings:
  - "list-network-resources: GREEN — read-only enumeration"
  - "get-network-resource: GREEN — read-only resource details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "power cycle radio unit: YELLOW — causes temporary service interruption"
```

## Escalation Conditions

- Radio unit fails SAS registration after multiple attempts over 2 hours
- Radio unit hardware appears physically damaged
- Multiple radio units at the same site fail to activate simultaneously
- Radio unit shows DECOMMISSIONED status unexpectedly

## Data Sensitivity

- `aws private-networks list-network-resources`: LOW — returns resource metadata
- `aws private-networks get-network-resource`: LOW — returns resource configuration
- Radio unit serial numbers: MEDIUM — hardware identifiers

## Prohibited Actions

- NEVER attempt to open or modify radio unit hardware
- NEVER operate radio units without proper CBRS authorization
- NEVER move radio units to a different location without updating the network site configuration
- NEVER power cycle radio units during active device connections without warning users

## Phase 3 — Rollback

1. If a radio unit was deactivated during troubleshooting, re-activate the network site
2. If radio unit was power cycled, wait for SAS re-registration (up to 30 minutes)
3. Verify all previously connected devices can reconnect after radio unit recovery
4. Check CloudWatch metrics to confirm radio unit is broadcasting normally
5. Document the activation issue and resolution for future reference

## Common Issues

- symptoms: "Radio unit status shows PROVISIONING for extended period"
  diagnosis: "Radio unit may not be physically installed or powered on."
  resolution: "Verify physical installation: power, ethernet, antennas. Confirm unit has internet connectivity."

- symptoms: "Radio unit shows ACTIVE but no devices can connect"
  diagnosis: "SAS registration may have succeeded but SIM provisioning is incomplete."
  resolution: "Check SIM status for connected devices. Verify SIMs are ACTIVE."

## Output Format

```yaml
root_cause: "radio_unit_activation — <specific_cause>"
evidence:
  - type: radio_status
    content: "<status details>"
severity: CRITICAL
mitigation:
  immediate: "Resolve activation blocker"
  long_term: "Document installation checklist for future deployments"
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
