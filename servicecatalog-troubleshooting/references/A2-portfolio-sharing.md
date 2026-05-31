---
title: "A2 — Portfolio Sharing"
description: "Diagnose Service Catalog portfolio sharing failures"
status: active
severity: HIGH
triggers: ["cannot share portfolio", "portfolio sharing failed", "imported portfolio not visible", "cross-account portfolio"]
owner: devops-agent
objective: "Resolve portfolio sharing issues across accounts and organizations"
context: "Portfolios can be shared via account ID, organization, or OU. Organization sharing requires Organizations integration. Imported portfolios are read-only."
---

## Phase 1 — Triage

MUST:
- Check existing shares: `aws servicecatalog list-portfolio-access --portfolio-id <portfolio-id>`
- For organization sharing, verify integration: `aws servicecatalog get-aws-organizations-access-status`
- In recipient account, check imported portfolios: `aws servicecatalog list-accepted-portfolio-shares`

SHOULD:
- Verify the target account ID or organization ID is correct
- Check if portfolio has products before sharing (empty portfolios are valid but useless)

MAY:
- Review CloudTrail for CreatePortfolioShare errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePortfolioShare --max-results 10`

## Phase 2 — Remediate

MUST:
- For organization sharing, enable access: `aws servicecatalog enable-aws-organizations-access`
- Share portfolio: `aws servicecatalog create-portfolio-share --portfolio-id <id> --account-id <account>` or `--organization-node Type=ORGANIZATION,Value=<org-id>`
- In recipient account, accept share: `aws servicecatalog accept-portfolio-share --portfolio-id <portfolio-id>`

SHOULD:
- Add launch constraints in the source account before sharing
- Verify products are visible in the recipient account after sharing

MAY:
- Use organization sharing for broad distribution instead of per-account sharing
- Automate portfolio sharing with CloudFormation custom resources

## Safety Ratings

```
safety_ratings:
  - "list-portfolio-access: GREEN — read-only share listing"
  - "create-portfolio-share: YELLOW — grants portfolio access to another account"
  - "enable-aws-organizations-access: YELLOW — enables org-wide integration"
  - "delete-portfolio-share: RED — removes access, may affect provisioned products"
```

## Escalation Conditions

- Organization sharing requires management account access
- Recipient account cannot see shared portfolio after accepting
- Portfolio sharing blocked by SCP in recipient account
- Shared portfolio products fail to provision in recipient account

## Data Sensitivity

- `aws servicecatalog list-portfolio-access`: MEDIUM — reveals account IDs with access
- `aws servicecatalog get-aws-organizations-access-status`: LOW — integration status
- `aws servicecatalog create-portfolio-share`: MEDIUM — grants cross-account access
- `aws cloudtrail lookup-events`: MEDIUM — user activity records

## Prohibited Actions

- NEVER share portfolios containing sensitive products to untrusted accounts
- NEVER enable organization access without management account approval
- NEVER delete portfolio shares without verifying no active provisioned products in recipient
- NEVER assume imported portfolios can be modified in the recipient account

## Phase 3 — Rollback

1. If portfolio was shared to wrong account: `aws servicecatalog delete-portfolio-share --portfolio-id <id> --account-id <wrong-account>`
2. If organization access was enabled incorrectly: `aws servicecatalog disable-aws-organizations-access`
3. If recipient accepted wrong portfolio: reject and remove from local principals
4. Verify provisioned products in recipient accounts are not affected
5. Document sharing changes and notify affected account owners

## Common Issues

- symptoms: "Shared portfolio is not visible in the recipient account"
  diagnosis: "Portfolio share was not accepted, or recipient lacks servicecatalog:AcceptPortfolioShare permission."
  resolution: "In recipient account: aws servicecatalog accept-portfolio-share --portfolio-id <id>. Ensure recipient has ServiceCatalog end user permissions."

## Output Format

```yaml
root_cause: "portfolio_sharing — <specific_cause>"
evidence:
  - type: sharing_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix portfolio sharing configuration"
  long_term: "Use organization sharing for scalable portfolio distribution"
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
