---
title: "A1 — Container Creation Failures"
description: "Diagnose MediaStore container creation failures"
status: active
severity: HIGH
triggers: ["container creation failed", "cannot create container", "mediastore container error"]
owner: devops-agent
objective: "Resolve MediaStore container creation failures"
context: "MediaStore containers are the top-level storage unit. Container names must be unique within an account and region."
---

## Phase 1 — Triage

MUST:
- Check existing containers: `aws mediastore list-containers --query 'Containers[*].{Name:Name,Status:Status}'`
- Verify IAM permissions for mediastore:CreateContainer
- Check if container name already exists (names must be unique per account/region)

SHOULD:
- Review CloudTrail for CreateContainer errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateContainer --max-results 10`
- Check service quotas: `aws service-quotas get-service-quota --service-code mediastore --quota-code L-3DAB5B8E`

MAY:
- Verify no SCP blocks MediaStore operations

## Phase 2 — Remediate

MUST:
- Fix IAM permissions: ensure mediastore:CreateContainer is allowed
- Use a unique container name (3-255 characters, lowercase letters, numbers, hyphens)
- Wait for container status to become ACTIVE before using

SHOULD:
- Tag containers for identification: `aws mediastore tag-resource --resource <arn> --tags Key=Environment,Value=production`
- Enable access logging at creation time

MAY:
- Automate container creation with CloudFormation or Terraform

## Safety Ratings

```
safety_ratings:
  - "mediastore list-containers: GREEN — read-only container enumeration"
  - "mediastore describe-container: GREEN — read-only container details"
  - "mediastore create-container: YELLOW — creates new storage resource"
  - "cloudtrail lookup-events: GREEN — read-only event query"
```

## Escalation Conditions

- Container creation fails with internal service errors after IAM fixes
- Service quota for containers has been reached
- Container creation succeeds but status remains CREATING for over 10 minutes
- SCP or permission boundary blocks MediaStore operations

## Data Sensitivity

- `aws mediastore list-containers`: LOW — returns container names and status
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and API actions
- Container ARN: LOW — resource identifier
- Service quota information: LOW — account limits

## Prohibited Actions

- NEVER create containers with overly permissive policies as a troubleshooting step
- NEVER delete existing containers to free up the name without verifying they are empty
- NEVER create containers in regions where they are not needed (incurs costs)
- NEVER share container endpoints publicly without a restrictive container policy

## Phase 3 — Rollback

1. Delete incorrectly created containers (must be empty): `aws mediastore delete-container --container-name <name>`
2. If container has objects, delete objects first: `aws mediastore-data delete-object --endpoint <endpoint> --path <path>`
3. Revert IAM policy changes if they were too permissive
4. Remove tags if they were applied incorrectly
5. Document the creation failure cause

## Common Issues

- symptoms: "CreateContainer returns ContainerInUseException"
  diagnosis: "A container with the same name already exists in this account and region."
  resolution: "Use a different container name or delete the existing container if it is no longer needed."

- symptoms: "Container status stays in CREATING state indefinitely"
  diagnosis: "Internal service issue or eventual consistency delay."
  resolution: "Wait up to 10 minutes. If still CREATING, delete and recreate. Contact AWS Support if persistent."

## Output Format

```yaml
root_cause: "container_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation parameters or permissions"
  long_term: "Automate container creation in IaC"
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
