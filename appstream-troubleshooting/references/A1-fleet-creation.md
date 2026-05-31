---
title: "A1 — Fleet Creation Failures"
description: "Diagnose AppStream 2.0 fleet creation failures"
status: active
severity: HIGH
triggers:
  - "fleet creation failed"
  - "cannot create fleet"
  - "fleet stuck in starting"
  - "fleet error"
owner: devops-agent
objective: "Identify why fleet creation is failing and resolve the issue"
context: "Fleet creation can fail due to IAM permissions, invalid image, VPC/subnet issues, instance type unavailability, service quotas, or security group misconfiguration. Fleets require a valid image and proper networking."
---

## Phase 1 — Triage

MUST:
- Check fleet status: `aws appstream describe-fleets --names <fleet-name>`
- Verify the image exists and is available: `aws appstream describe-images --names <image-name>`
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names appstream:CreateFleet`
- Check service quotas: `aws service-quotas get-service-quota --service-code appstream2 --quota-code L-5765E346`

SHOULD:
- Verify VPC subnets exist and are in different AZs (if using VPC): `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check security groups: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify the instance type is available in the region
- Check CloudTrail for fleet creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFleet --max-results 10`

MAY:
- Check if the image platform matches the fleet type (Windows/Linux)
- Verify the IAM service role for AppStream

## Phase 2 — Remediate

MUST:
- Use a valid, available image for the fleet
- Fix IAM permissions for fleet creation
- Ensure VPC subnets are in at least two different AZs

SHOULD:
- Create fleet with proper configuration:
  `aws appstream create-fleet --name <fleet-name> --image-name <image-name> --instance-type <type> --fleet-type ON_DEMAND --compute-capacity DesiredInstances=1 --vpc-config SubnetIds=<subnet1>,<subnet2>,SecurityGroupIds=<sg-id>`
- Configure appropriate instance type for the applications

MAY:
- Request quota increase for fleet instances
- Tag the fleet for cost allocation

## Common Issues

- symptoms: "Fleet stuck in STARTING state"
  diagnosis: "VPC networking issue or instance type unavailable."
  resolution: "Check VPC subnets, security groups, and NAT gateway. Try a different instance type."

- symptoms: "InvalidParameterCombination error"
  diagnosis: "Image platform doesn't match fleet configuration."
  resolution: "Ensure Windows image with Windows fleet, Linux image with Linux fleet."

- symptoms: "LimitExceededException"
  diagnosis: "Fleet or instance quota exceeded."
  resolution: "Request quota increase or delete unused fleets."

## Output Format

```yaml
root_cause: "fleet_creation_failure — <specific_cause>"
evidence:
  - type: fleet_status
    content: "<fleet state and errors>"
  - type: image_status
    content: "<image availability>"
  - type: vpc_config
    content: "<VPC subnet and security group status>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Automate fleet creation with IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use a valid, available image | GREEN | Image selection |
| Fix IAM permissions | YELLOW | IAM policy change |
| Ensure VPC subnets in different AZs | GREEN | Configuration verification |
| Create fleet with proper configuration | YELLOW | New fleet creation |
| Configure appropriate instance type | GREEN | Instance selection |
| Request quota increase | GREEN | Quota management |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Fleet configuration**: Instance type, VPC, image details — INTERNAL
- **IAM permissions**: Fleet creation roles — INTERNAL
- **VPC configuration**: Subnet and security group details — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER create fleets with public subnets without security review
- NEVER use overly permissive security groups for fleet instances

## Phase 3 — Rollback

1. Delete the newly created fleet if it caused issues: `aws appstream delete-fleet --name <fleet-name>`
2. Revert IAM policy changes if permissions were modified
3. Restore previous fleet configuration if an update was made
4. Verify existing fleets are unaffected
5. Communicate fleet changes to affected users

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
