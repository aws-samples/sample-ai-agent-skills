---
title: "B1 — SaaS Integration Issues"
description: "Diagnose SaaS product integration and fulfillment issues"
status: active
severity: HIGH
triggers: ["saas not working", "saas registration failed", "redirect error", "saas fulfillment"]
owner: devops-agent
objective: "Resolve SaaS product integration issues between Marketplace and seller fulfillment"
context: "SaaS products require post-subscribe registration on the seller's website. The buyer clicks a redirect URL after subscribing to complete setup."
---

## Phase 1 — Triage

MUST:
- Verify subscription exists: `aws marketplace-entitlement get-entitlements --product-code <product-code>`
- Check subscription status in Marketplace console: Manage Subscriptions
- Verify the SaaS redirect URL was followed after subscription

SHOULD:
- Check CloudTrail for subscription events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Subscribe --max-results 10`
- Verify entitlement dimensions match expected access level

MAY:
- Contact seller support if redirect URL fails
- Check browser network logs for redirect errors

## Phase 2 — Remediate

MUST:
- Complete SaaS registration by following the redirect URL from Marketplace console
- Verify entitlements are active and not expired: `aws marketplace-entitlement get-entitlements --product-code <code> --filter '{"CUSTOMER_IDENTIFIER":["<customer-id>"]}'`
- Contact seller if registration page returns errors

SHOULD:
- Verify the AWS account ID matches between Marketplace subscription and seller registration
- Check if the seller requires additional IAM roles or permissions

MAY:
- Set up SSO integration if the seller supports it
- Configure API access tokens from the seller's portal

## Safety Ratings

```
safety_ratings:
  - "marketplace-entitlement get-entitlements: GREEN — read-only entitlement check"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "SaaS registration on seller website: YELLOW — creates account on third-party system"
  - "IAM role creation for seller: YELLOW — grants cross-account access"
```

## Escalation Conditions

- Seller's registration page is down or returns persistent errors
- Entitlements show active but seller denies access
- Cross-account IAM role setup fails with trust policy errors
- SaaS product requires VPC connectivity not available

## Data Sensitivity

- `aws marketplace-entitlement get-entitlements`: MEDIUM — reveals subscription details
- Seller registration: HIGH — may require sharing account details with third party
- IAM cross-account roles: HIGH — grants external access to AWS resources
- SaaS credentials: HIGH — authentication tokens for third-party service

## Prohibited Actions

- NEVER share AWS account credentials with the seller
- NEVER create overly permissive IAM roles for seller integration
- NEVER bypass seller registration by attempting direct API access
- NEVER store SaaS credentials in plaintext or version control

## Phase 3 — Rollback

1. If SaaS registration created unwanted accounts, contact seller to deactivate
2. If IAM roles were created for seller integration, delete them: `aws iam delete-role --role-name <role-name>`
3. If entitlements are incorrect, contact AWS Support to reconcile
4. Revoke any API tokens or credentials issued during troubleshooting
5. Document the integration issue for future reference

## Common Issues

- symptoms: "Subscribed but cannot access SaaS product"
  diagnosis: "Post-subscribe registration on seller website was not completed."
  resolution: "Go to Marketplace console > Manage Subscriptions > Set up product to follow the redirect URL."

## Output Format

```yaml
root_cause: "saas_integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Complete SaaS registration workflow"
  long_term: "Document SaaS onboarding process for team"
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
