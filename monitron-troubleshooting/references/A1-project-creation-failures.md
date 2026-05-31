---
title: "A1 — Project Creation Failures"
description: "Diagnose Monitron project creation and configuration failures"
status: active
severity: HIGH
triggers: ["cannot create project", "project creation failed", "CreateProject error"]
owner: devops-agent
objective: "Resolve Monitron project creation issues"
context: "Monitron projects are the top-level container for gateways, sensors, and assets. Projects require IAM Identity Center integration and proper IAM permissions."
---

## Phase 1 — Triage

MUST:
- List existing projects: `aws monitron list-projects --query 'projectSummaries[*].{Name:name,Arn:projectArn,CreatedAt:createdAt}'`
- Check IAM permissions for monitron:CreateProject: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names monitron:CreateProject`
- Verify IAM Identity Center is enabled: `aws sso-admin list-instances`

SHOULD:
- Check CloudTrail for CreateProject errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateProject --max-results 10`
- Verify the region supports Monitron

MAY:
- Check service quotas for Monitron projects: `aws service-quotas get-service-quota --service-code monitron --quota-code L-XXXXXXXX`
- Review organization SCPs for monitron restrictions

## Phase 2 — Remediate

MUST:
- Ensure IAM Identity Center is enabled in the account
- Grant monitron:CreateProject permission to the calling principal
- Verify the project name is unique within the account and region

SHOULD:
- Set up IAM Identity Center before creating Monitron projects
- Configure admin users during project creation

MAY:
- Use CloudFormation to create projects with consistent configuration
- Tag projects for cost allocation and organization

## Common Issues

- symptoms: "CreateProject returns AccessDeniedException"
  diagnosis: "Calling principal lacks monitron:CreateProject permission."
  resolution: "Add monitron:CreateProject to the IAM policy. Verify no SCP blocks monitron actions."

- symptoms: "Project creation fails with SSO not configured error"
  diagnosis: "IAM Identity Center is not enabled in the account."
  resolution: "Enable IAM Identity Center in the account. This is a prerequisite for Monitron."

## Safety Ratings

```
safety_ratings:
  - "monitron list-projects: GREEN — read-only enumeration"
  - "monitron get-project: GREEN — read-only detail retrieval"
  - "monitron create-project: YELLOW — creates a new project resource"
  - "monitron delete-project: RED — permanently removes project and all associated data"
```

## Escalation Conditions

- Project creation fails with internal service errors after fixing permissions
- IAM Identity Center setup requires organization-level changes
- Service quota limit reached for Monitron projects
- Region does not support Monitron service

## Data Sensitivity

- `aws monitron list-projects`: LOW — returns project metadata only
- `aws sso-admin list-instances`: LOW — returns SSO instance metadata
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities

## Prohibited Actions

- NEVER delete an existing project to resolve creation issues — data loss is permanent
- NEVER disable IAM Identity Center to troubleshoot — it affects all SSO-dependent services
- NEVER create projects in unsupported regions
- NEVER share project ARNs or admin user details outside the organization

## Phase 3 — Rollback

1. If a project was created with wrong configuration, delete it: `aws monitron delete-project --project-id <id>` (only if no gateways or sensors are registered)
2. If IAM policies were modified, restore previous versions
3. If IAM Identity Center was configured incorrectly, consult the SSO admin
4. Verify no other services were affected by IAM changes
5. Document the creation failure and resolution

## Output Format

```yaml
root_cause: "project_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from API or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix permissions and prerequisites"
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
