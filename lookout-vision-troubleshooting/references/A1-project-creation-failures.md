---
title: "A1 — Project Creation Failures"
description: "Diagnose Lookout for Vision project creation and configuration failures"
status: active
severity: HIGH
triggers: ["cannot create project", "project creation failed", "CreateProject error"]
owner: devops-agent
objective: "Resolve project creation issues and establish the project for dataset import"
context: "Projects are the top-level container for datasets and models. They require IAM permissions and a unique project name within the account and region."
---

## Phase 1 — Triage

MUST:
- List existing projects: `aws lookoutvision list-projects --query 'Projects[*].{Name:ProjectName,Arn:ProjectArn}'`
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names lookoutvision:CreateProject`
- Check CloudTrail for errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateProject --max-results 10`

SHOULD:
- Verify the project name is unique within the account and region
- Check service quotas for maximum projects

MAY:
- Review organization SCPs for lookoutvision restrictions
- Check if the region supports Lookout for Vision

## Phase 2 — Remediate

MUST:
- Grant lookoutvision:CreateProject permission to the calling principal
- Use a unique project name within the account and region
- Verify the region supports Lookout for Vision

SHOULD:
- Use descriptive project names following naming conventions
- Tag projects for cost allocation

MAY:
- Use CloudFormation or Terraform for project creation
- Set up project naming conventions for the team

## Common Issues

- symptoms: "CreateProject returns AccessDeniedException"
  diagnosis: "Calling principal lacks lookoutvision:CreateProject permission."
  resolution: "Add lookoutvision:CreateProject to the IAM policy. Check for SCP restrictions."

- symptoms: "Project name already exists error"
  diagnosis: "A project with the same name exists in this account and region."
  resolution: "Use a different project name or delete the existing project if it's unused."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision list-projects: GREEN — read-only enumeration"
  - "lookoutvision create-project: YELLOW — creates a new project resource"
  - "lookoutvision delete-project: RED — permanently removes project and all models"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
```

## Escalation Conditions

- Project creation fails with internal service errors after fixing permissions
- Service quota limit reached for projects
- Region does not support Lookout for Vision
- SCP blocks lookoutvision actions at organization level

## Data Sensitivity

- `aws lookoutvision list-projects`: LOW — returns project metadata only
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities
- Project names: LOW — organizational identifiers

## Prohibited Actions

- NEVER delete an existing project to resolve naming conflicts without checking for active models
- NEVER grant lookoutvision:* wildcard without scoping to specific resources
- NEVER create projects in unsupported regions
- NEVER share project ARNs outside the organization

## Phase 3 — Rollback

1. If a project was created with wrong name, delete it: `aws lookoutvision delete-project --project-name <name>` (only if no datasets or models exist)
2. If IAM policies were modified, restore previous versions
3. Verify no other resources were affected by IAM changes
4. Document the creation failure and resolution
5. Confirm the project is accessible after creation

## Output Format

```yaml
root_cause: "project_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from API or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix permissions and retry creation"
  long_term: "Use IaC for project creation with pre-validated configuration"
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
