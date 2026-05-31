---
title: "A1 — Portfolio Creation"
description: "Diagnose Service Catalog portfolio creation failures"
status: active
severity: HIGH
triggers: ["cannot create portfolio", "portfolio creation failed", "portfolio error"]
owner: devops-agent
objective: "Resolve portfolio creation and configuration issues"
context: "Portfolios are containers for products. They require admin permissions and can be shared across accounts or organizations."
---

## Phase 1 — Triage

MUST:
- Check existing portfolios: `aws servicecatalog list-portfolios --query 'PortfolioDetails[*].{Id:Id,Name:DisplayName}'`
- Verify IAM permissions for servicecatalog:CreatePortfolio: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names servicecatalog:CreatePortfolio`
- Check for duplicate portfolio names (allowed but confusing)

SHOULD:
- Review CloudTrail for CreatePortfolio errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePortfolio --max-results 10`
- Verify service-linked role exists if using organization features

MAY:
- Check service quotas for portfolios: `aws service-quotas get-service-quota --service-code servicecatalog --quota-code L-B0BEF860`

## Phase 2 — Remediate

MUST:
- Grant servicecatalog:CreatePortfolio and servicecatalog:TagResource permissions
- Provide required parameters: DisplayName and ProviderName
- Ensure tags comply with organization tag policies if applicable

SHOULD:
- Use consistent naming conventions for portfolios
- Associate portfolios with IAM principals immediately after creation

MAY:
- Automate portfolio creation with CloudFormation or Terraform
- Set up portfolio sharing as part of creation workflow

## Safety Ratings

```
safety_ratings:
  - "list-portfolios: GREEN — read-only enumeration"
  - "CreatePortfolio: YELLOW — creates a new portfolio resource"
  - "associate-principal-with-portfolio: YELLOW — grants access to users/roles"
  - "delete-portfolio: RED — removes portfolio and all associations"
```

## Escalation Conditions

- Portfolio creation blocked by SCP that operator cannot modify
- Service quota for portfolios reached and increase requires AWS support
- Organization-level portfolio sharing requires management account access
- Portfolio creation fails with internal service errors after permission fixes

## Data Sensitivity

- `aws servicecatalog list-portfolios`: LOW — portfolio metadata
- `aws iam simulate-principal-policy`: LOW — permission simulation
- `aws cloudtrail lookup-events`: MEDIUM — user activity records
- Portfolio details: LOW — contains names and provider info

## Prohibited Actions

- NEVER delete portfolios with active provisioned products
- NEVER share portfolios to untrusted accounts without approval
- NEVER create portfolios with overly permissive principal associations
- NEVER use AdministratorAccess as the portfolio admin role

## Phase 3 — Rollback

1. If portfolio was created with wrong configuration: `aws servicecatalog update-portfolio --id <portfolio-id> --display-name <correct-name>`
2. If portfolio was shared incorrectly: `aws servicecatalog delete-portfolio-share --portfolio-id <id> --account-id <account>`
3. If wrong principals were associated: `aws servicecatalog disassociate-principal-from-portfolio --portfolio-id <id> --principal-arn <arn>`
4. If portfolio needs to be deleted: remove all products and associations first
5. Document portfolio changes and notify affected teams

## Common Issues

- symptoms: "CreatePortfolio returns AccessDeniedException"
  diagnosis: "IAM user/role lacks servicecatalog:CreatePortfolio permission."
  resolution: "Attach AWSServiceCatalogAdminFullAccess managed policy or add specific servicecatalog:Create* permissions."

## Output Format

```yaml
root_cause: "portfolio_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix portfolio creation issue"
  long_term: "Automate portfolio management with IaC"
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
