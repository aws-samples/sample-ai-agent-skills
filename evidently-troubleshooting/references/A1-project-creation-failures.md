---
title: "A1 — Project Creation Failures"
description: "Diagnose Evidently project creation failures"
status: active
severity: HIGH
triggers: ["cannot create project", "project creation failed", "CreateProject error"]
owner: devops-agent
objective: "Resolve project creation issues for CloudWatch Evidently"
context: "Projects are the top-level container for features, launches, and experiments. They require unique names within a region."
---

## Phase 1 — Triage

MUST:
- Check existing projects: `aws evidently list-projects`
- Verify IAM permissions for evidently:CreateProject
- Confirm project name is unique within the account and region

SHOULD:
- Check service quotas for maximum projects per account
- Verify the project name meets naming requirements (alphanumeric, hyphens, underscores)

MAY:
- Review CloudTrail for CreateProject errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=evidently.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include evidently:CreateProject
- Use a unique project name within the region
- Ensure project name follows naming conventions

SHOULD:
- Configure data delivery during project creation for evaluation data export
- Tag the project for cost allocation and organization

MAY:
- Set up project via IaC (CloudFormation/Terraform) for reproducibility
- Configure data delivery to S3 or CloudWatch Logs during creation

## Common Issues

- symptoms: "CreateProject returns ConflictException"
  diagnosis: "A project with the same name already exists in this region."
  resolution: "List existing projects: `aws evidently list-projects`. Use a different name or update the existing project."

## Safety Ratings

```
safety_ratings:
  - "list-projects: GREEN — read-only enumeration of existing projects"
  - "CreateProject: YELLOW — creates a new project resource"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Project creation fails with internal service errors after fixing all input parameters
- Service quota limit reached and quota increase request is needed
- Project creation blocked by SCP or permission boundary
- Multiple projects needed across regions for a coordinated deployment

## Data Sensitivity

- `aws evidently list-projects`: LOW — returns project metadata only
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions

## Prohibited Actions

- NEVER delete an existing project to reuse its name without explicit approval
- NEVER create projects without considering data delivery configuration
- NEVER ignore ConflictException — it indicates a naming collision
- NEVER create projects in regions where they are not needed

## Phase 3 — Rollback

1. If a project was created with incorrect configuration, update it: `aws evidently update-project --project <name>`
2. If the project should not exist, delete it: `aws evidently delete-project --project <name>` (only if confirmed empty and not needed)
3. If IAM policy changes were made, revert to the previous policy version
4. Verify no other projects were affected by IAM or quota changes
5. Document the creation attempt and outcome

## Output Format

```yaml
root_cause: "project_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation parameters"
  long_term: "Deploy projects via IaC with validated configurations"
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
