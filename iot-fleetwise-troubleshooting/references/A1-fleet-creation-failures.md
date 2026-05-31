---
title: "A1 — Fleet Creation Failures"
description: "Diagnose fleet creation and vehicle association failures"
status: active
severity: HIGH
triggers: ["fleet creation failed", "cannot create fleet", "fleet association error", "fleet not found"]
owner: devops-agent
objective: "Resolve fleet creation failures and vehicle-fleet association issues"
context: "Fleets group vehicles for campaign targeting. Vehicles must be explicitly associated with fleets. Fleet IDs must be unique within the account and region."
---

## Phase 1 — Triage

MUST:
- List existing fleets: `aws iotfleetwise list-fleets --query 'fleetSummaries[*].{Id:id,Arn:arn}'`
- Check if fleet already exists: `aws iotfleetwise get-fleet --fleet-id <id>`
- Verify IAM permissions for iotfleetwise:CreateFleet

SHOULD:
- Check signal catalog exists (required dependency): `aws iotfleetwise list-signal-catalogs`
- Verify the fleet's signal catalog ARN is valid
- Review CloudTrail for CreateFleet errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFleet --max-results 10`

MAY:
- Check service quotas for maximum fleets per account
- Verify no naming conflicts with existing fleet IDs

## Phase 2 — Remediate

MUST:
- Ensure signal catalog exists before creating fleet
- Fix IAM permissions: iotfleetwise:CreateFleet, iotfleetwise:AssociateVehicleFleet
- Use unique fleet ID that does not conflict with existing fleets

SHOULD:
- Associate vehicles with fleet after creation: `aws iotfleetwise associate-vehicle-fleet --vehicle-name <name> --fleet-id <id>`
- Verify associated vehicles use compatible model manifests
- Tag fleets for organization: `aws iotfleetwise tag-resource --resource-arn <arn> --tags Key=Environment,Value=Production`

MAY:
- Create fleets via CloudFormation for reproducibility
- Set up fleet naming conventions for multi-team environments

## Safety Ratings

```
safety_ratings:
  - "list-fleets: GREEN — read-only enumeration"
  - "get-fleet: GREEN — read-only inspection"
  - "create-fleet: YELLOW — creates fleet resource"
  - "associate-vehicle-fleet: YELLOW — adds vehicle to fleet, affects campaign targeting"
  - "delete-fleet: RED — removes fleet and all vehicle associations"
```

## Escalation Conditions

- Fleet creation fails with internal service errors after permission fixes
- Fleet association fails for vehicles that should be compatible
- Fleet quota limits reached requiring account-level increase
- Fleet deletion needed but has active campaigns targeting it

## Data Sensitivity

- `aws iotfleetwise list-fleets`: LOW — returns fleet metadata
- `aws iotfleetwise get-fleet`: LOW — returns fleet configuration
- `aws iotfleetwise list-vehicles-in-fleet`: MEDIUM — returns vehicle identifiers
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER delete a fleet with active campaigns without suspending campaigns first
- NEVER disassociate vehicles from a fleet during active data collection
- NEVER create fleets without a signal catalog — it is a required dependency
- NEVER assume fleet membership is automatic — always explicitly associate vehicles

## Phase 3 — Rollback

1. If fleet was created in error, disassociate all vehicles first: `aws iotfleetwise disassociate-vehicle-fleet --vehicle-name <name> --fleet-id <id>`
2. Delete the fleet: `aws iotfleetwise delete-fleet --fleet-id <id>`
3. If vehicles were associated with wrong fleet, disassociate and re-associate with correct fleet
4. Verify campaigns are not affected by fleet changes
5. Document the rollback and notify the vehicle data team

## Common Issues

- symptoms: "CreateFleet returns ResourceNotFoundException for signal catalog"
  diagnosis: "Signal catalog does not exist in the account/region. FleetWise requires a signal catalog as a foundational resource."
  resolution: "Create a signal catalog first: aws iotfleetwise create-signal-catalog --name <name> --nodes <signal-definitions>"

- symptoms: "AssociateVehicleFleet returns ValidationException"
  diagnosis: "Vehicle does not exist or fleet ID is invalid."
  resolution: "Verify vehicle exists: aws iotfleetwise get-vehicle --vehicle-name <name>. Verify fleet exists: aws iotfleetwise get-fleet --fleet-id <id>."

## Output Format

```yaml
root_cause: "fleet_creation — <specific_cause>"
evidence:
  - type: fleet_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix fleet configuration"
  long_term: "Automate fleet creation via IaC"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
