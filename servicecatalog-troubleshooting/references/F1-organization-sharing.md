---
title: "F1 — Organization Sharing"
description: "Diagnose Service Catalog organization-level portfolio sharing issues"
status: active
severity: HIGH
triggers: ["organization sharing", "org sharing failed", "portfolio not visible in member account", "organizations integration"]
owner: devops-agent
objective: "Resolve organization-level portfolio sharing and integration issues"
context: "Organization sharing distributes portfolios to all accounts in an organization or OU. Requires Organizations integration enabled. Shared portfolios are read-only in recipient accounts."
---

## Phase 1 — Triage

MUST:
- Check organization access status: `aws servicecatalog get-aws-organizations-access-status`
- Verify organization structure: `aws organizations describe-organization`
- List organization shares: `aws servicecatalog list-organization-portfolio-access --portfolio-id <portfolio-id>`

SHOULD:
- Verify caller is in management account or delegated admin
- Check if target OU exists: `aws organizations list-organizational-units-for-parent --parent-id <parent-id>`

MAY:
- Review CloudTrail for organization sharing events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePortfolioShare --max-results 10`
- Check if SCPs block Service Catalog in member accounts

## Phase 2 — Remediate

MUST:
- Enable organization access: `aws servicecatalog enable-aws-organizations-access`
- Share portfolio to organization: `aws servicecatalog create-portfolio-share --portfolio-id <id> --organization-node Type=ORGANIZATION,Value=<org-id>`
- Or share to specific OU: `aws servicecatalog create-portfolio-share --portfolio-id <id> --organization-node Type=ORGANIZATIONAL_UNIT,Value=<ou-id>`

SHOULD:
- Verify portfolio is visible in member accounts after sharing
- Set up launch constraints before sharing to ensure provisioning works

MAY:
- Use delegated administrator for Service Catalog management
- Automate organization sharing with CloudFormation

## Safety Ratings

```
safety_ratings:
  - "get-aws-organizations-access-status: GREEN — read-only status check"
  - "enable-aws-organizations-access: YELLOW — enables org-wide integration"
  - "create-portfolio-share (organization): YELLOW — shares to all accounts in org/OU"
  - "disable-aws-organizations-access: RED — disables org integration, affects all shares"
```

## Escalation Conditions

- Organization access cannot be enabled without management account access
- SCPs in member accounts block Service Catalog operations
- Portfolio sharing fails due to organization structure changes
- Delegated administrator setup requires management account action

## Data Sensitivity

- `aws organizations describe-organization`: MEDIUM — organization structure
- `aws servicecatalog list-organization-portfolio-access`: MEDIUM — reveals sharing scope
- `aws organizations list-organizational-units-for-parent`: LOW — OU structure
- `aws cloudtrail lookup-events`: MEDIUM — sharing activity records

## Prohibited Actions

- NEVER enable organization access without management account owner approval
- NEVER share portfolios containing sensitive products to the entire organization
- NEVER disable organization access without understanding impact on existing shares
- NEVER modify organization structure to fix sharing issues

## Phase 3 — Rollback

1. If portfolio was shared to wrong OU: `aws servicecatalog delete-portfolio-share --portfolio-id <id> --organization-node Type=ORGANIZATIONAL_UNIT,Value=<wrong-ou-id>`
2. If organization access was enabled incorrectly: `aws servicecatalog disable-aws-organizations-access` (affects all org shares)
3. If delegated admin was set up incorrectly: deregister from management account
4. Verify provisioned products in member accounts are not affected
5. Document sharing changes and notify affected account owners

## Common Issues

- symptoms: "create-portfolio-share with organization-node fails with 'Organizations access is not enabled'"
  diagnosis: "Service Catalog Organizations integration has not been enabled."
  resolution: "Enable from management account: aws servicecatalog enable-aws-organizations-access. Then retry the share."

## Output Format

```yaml
root_cause: "organization_sharing — <specific_cause>"
evidence:
  - type: sharing_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix organization sharing configuration"
  long_term: "Establish organization sharing governance with delegated admin"
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
