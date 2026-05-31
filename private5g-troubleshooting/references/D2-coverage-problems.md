---
title: "D2 — Coverage Problems"
description: "Diagnose coverage gaps and dead zones"
status: active
severity: HIGH
triggers: ["no coverage", "dead zone", "signal loss", "coverage gap"]
owner: devops-agent
objective: "Resolve coverage gaps and signal issues in the Private 5G deployment"
context: "Coverage depends on radio unit count, placement, antenna orientation, physical environment, and CBRS power limits."
---

## Phase 1 — Triage

MUST:
- List radio units and their status: `aws private-networks list-network-resources --network-arn <arn> --filters name=type,values=RADIO_UNIT`
- Verify all radio units are ACTIVE: `aws private-networks get-network-resource --network-resource-arn <arn>`
- Check connected device count to identify coverage patterns: `aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name ConnectedDevices --dimensions Name=NetworkArn,Value=<arn> --start-time <start> --end-time <end> --period 3600 --statistics Maximum`

SHOULD:
- Map reported dead zones against radio unit locations
- Check if any radio units went offline recently
- Verify antenna connections on radio units

MAY:
- Conduct a signal strength survey at the affected location
- Check for new physical obstacles (construction, equipment) blocking signal

## Phase 2 — Remediate

MUST:
- Restore any offline radio units to ACTIVE status
- Verify antenna connections are secure on all radio units
- Confirm radio units have proper power supply

SHOULD:
- Add radio units to cover identified dead zones
- Reposition existing radio units for better coverage overlap
- Adjust antenna orientation for optimal coverage pattern

MAY:
- Engage AWS Professional Services for RF site survey
- Plan phased radio unit deployment for large facilities

## Safety Ratings

```
safety_ratings:
  - "list-network-resources: GREEN — read-only enumeration"
  - "get-network-resource: GREEN — read-only resource details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "radio unit repositioning: YELLOW — temporary coverage disruption"
```

## Escalation Conditions

- Coverage gaps persist after adding radio units
- Radio units are active but signal strength is abnormally low
- Coverage area shrinks over time without physical changes
- SAS authorization issues limiting radio unit transmit power

## Data Sensitivity

- `aws private-networks list-network-resources`: LOW — returns resource metadata
- Coverage survey data: LOW — RF signal measurements
- Facility layout information: MEDIUM — physical security implications

## Prohibited Actions

- NEVER attempt to increase radio unit transmit power beyond CBRS limits
- NEVER modify radio unit hardware or external antennas without authorization
- NEVER guarantee specific coverage distances — environment heavily affects range
- NEVER remove radio units from one area to cover another without impact analysis

## Phase 3 — Rollback

1. If radio units were repositioned, document original and new locations
2. If radio units were added, verify they do not cause interference with existing units
3. Restore original antenna orientations if changes worsened coverage
4. Verify coverage in previously working areas is not degraded
5. Document coverage map changes for facility management

## Common Issues

- symptoms: "Coverage gap in a specific area of the facility"
  diagnosis: "Radio unit coverage does not reach the area due to distance or obstacles."
  resolution: "Add a radio unit closer to the affected area. Remove or work around physical obstacles."

- symptoms: "Coverage was working but suddenly dropped"
  diagnosis: "Radio unit may have gone offline or lost SAS authorization."
  resolution: "Check radio unit status. Verify power and network connectivity. Check SAS registration."

## Output Format

```yaml
root_cause: "coverage — <specific_cause>"
evidence:
  - type: coverage_analysis
    content: "<coverage data>"
severity: HIGH
mitigation:
  immediate: "Restore coverage to affected area"
  long_term: "Comprehensive RF planning and radio unit deployment"
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
