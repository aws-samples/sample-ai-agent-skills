---
title: "A1 — AMG Workspace Creation Failures"
description: "Diagnose why an Amazon Managed Grafana workspace fails to create"
status: active
severity: HIGH
triggers:
  - "workspace creation failed"
  - "create workspace error"
  - "workspace stuck creating"
  - "CREATE_FAILED"
owner: devops-agent
objective: "Identify why an AMG workspace fails to create and resolve the blocking issue"
context: "AMG workspace creation requires IAM roles, authentication configuration, and optionally VPC settings. Failures typically stem from IAM permission issues, service-linked role problems, invalid VPC configuration, or account-level service limits."
---

## Phase 1 — Triage

MUST:
- Check workspace status: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.{Status:status,StatusReason:statusReason}'`
- List workspaces to confirm state: `aws grafana list-workspaces`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateWorkspace --max-results 10`
- Verify the IAM role exists and is assumable: `aws iam get-role --role-name <role-name>`
- Check the role trust policy allows grafana.amazonaws.com: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- Check service quotas for AMG: `aws service-quotas get-service-quota --service-code grafana --quota-code L-6F494529`
- Verify IAM Identity Center is enabled if using SSO authentication: `aws sso-admin list-instances`
- Check if the service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForAmazonGrafana`

MAY:
- Check VPC configuration if specified: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Verify security groups: `aws ec2 describe-security-groups --group-ids <sg-ids>`

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policy to include grafana.amazonaws.com as a trusted service
- Ensure the IAM role has the required permissions for configured data sources
- Create the service-linked role if missing: `aws iam create-service-linked-role --aws-service-name grafana.amazonaws.com`

SHOULD:
- Verify VPC subnets are in at least two Availability Zones
- Ensure security groups allow outbound HTTPS (443) traffic
- Request service quota increase if at the workspace limit

MAY:
- Retry workspace creation after fixing the identified issue
- Use CloudFormation or Terraform for repeatable workspace provisioning

## Common Issues

- symptoms: "Workspace status is CREATE_FAILED with permission error"
  diagnosis: "IAM role trust policy does not allow grafana.amazonaws.com to assume the role."
  resolution: "Update the role trust policy to include grafana.amazonaws.com as a trusted principal."

- symptoms: "Workspace creation fails with service-linked role error"
  diagnosis: "The AWSServiceRoleForAmazonGrafana service-linked role does not exist."
  resolution: "Create the service-linked role: aws iam create-service-linked-role --aws-service-name grafana.amazonaws.com"

- symptoms: "Workspace creation fails with VPC error"
  diagnosis: "Specified subnets are in the same AZ or security groups block required traffic."
  resolution: "Use subnets in at least two AZs and ensure security groups allow outbound HTTPS."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace: GREEN — read-only status check"
  - "list-workspaces: GREEN — read-only enumeration"
  - "create-service-linked-role: YELLOW — creates IAM role with service permissions"
  - "create VPC configuration: YELLOW — creates ENIs in specified subnets"
  - "retry workspace creation: YELLOW — provisions new managed infrastructure"
```

## Escalation Conditions

- Workspace creation fails repeatedly after IAM and VPC fixes
- Service-linked role creation blocked by SCP or permission boundary
- Workspace limit reached and quota increase is needed for production deployment
- VPC configuration requires cross-team network changes

## Data Sensitivity

- `aws grafana describe-workspace`: LOW — returns workspace metadata
- `aws iam get-role (trust policy)`: MEDIUM — reveals service trust relationships
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- VPC subnet and security group details: MEDIUM — reveals network architecture

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER create workspaces with overly permissive IAM roles (e.g., AdministratorAccess)
- NEVER configure VPC with public subnets unless explicitly required

## Phase 3 — Rollback

1. If a workspace was created with wrong configuration, delete it: `aws grafana delete-workspace --workspace-id <workspace-id>`
2. If a service-linked role was created in error, delete it: `aws iam delete-service-linked-role --role-name AWSServiceRoleForAmazonGrafana`
3. If VPC ENIs were created, they will be cleaned up when the workspace is deleted
4. Remove any IAM roles created specifically for the failed workspace
5. Document the failure and root cause for the next creation attempt

## Output Format

```yaml
root_cause: "workspace_creation_failure — <specific_cause>"
evidence:
  - type: workspace_status
    content: "<status and statusReason>"
  - type: iam_role
    content: "<role trust policy and permissions>"
  - type: cloudtrail
    content: "<creation event details>"
severity: HIGH
mitigation:
  immediate: "Fix the identified IAM, VPC, or quota issue and retry creation"
  long_term: "Use IaC templates with validated IAM roles and VPC configurations"
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
