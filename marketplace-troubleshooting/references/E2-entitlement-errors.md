---
title: "E2 — Entitlement Errors"
description: "Diagnose entitlement verification and enforcement errors"
status: active
severity: HIGH
triggers: ["entitlement error", "entitlement expired", "no entitlements", "entitlement check failed"]
owner: devops-agent
objective: "Resolve entitlement verification failures"
context: "Entitlements define usage dimensions and limits within a subscription. Sellers check entitlements via get-entitlements API to enforce access controls."
---

## Phase 1 — Triage

MUST:
- Check current entitlements: `aws marketplace-entitlement get-entitlements --product-code <product-code>`
- Verify entitlement dimensions and values: check ExpirationDate and Value fields
- Confirm subscription is active in Marketplace console

SHOULD:
- Check IAM permissions for entitlement API: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names aws-marketplace:GetEntitlements`
- Review application entitlement check logic for errors

MAY:
- Check if entitlements were recently modified by the buyer
- Verify entitlement caching behavior in the application

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for aws-marketplace:GetEntitlements
- Handle expired entitlements gracefully in application code
- Verify product code matches between application and Marketplace

SHOULD:
- Implement entitlement caching with appropriate TTL (5-15 minutes)
- Add fallback behavior when entitlement service is unavailable
- Set up SNS notifications for entitlement changes: `aws marketplace-entitlement get-entitlements --product-code <code> --filter '{"CUSTOMER_IDENTIFIER":["<id>"]}'`

MAY:
- Implement tiered access based on entitlement dimensions
- Set up monitoring for entitlement check failures

## Safety Ratings

```
safety_ratings:
  - "marketplace-entitlement get-entitlements: GREEN — read-only entitlement check"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "Application entitlement enforcement: YELLOW — affects customer access"
  - "Entitlement cache configuration: YELLOW — affects access control timing"
```

## Escalation Conditions

- Entitlements show empty for a customer with active subscription
- Entitlement service returns persistent errors
- Customer reports access denied despite valid entitlements
- Entitlement dimensions do not match product configuration

## Data Sensitivity

- `aws marketplace-entitlement get-entitlements`: MEDIUM — reveals customer subscription details
- Customer identifiers: HIGH — identifies specific buyers
- Entitlement values: MEDIUM — reveals subscription tier and limits
- Application access logs: MEDIUM — reveals customer usage patterns

## Prohibited Actions

- NEVER bypass entitlement checks to grant unauthorized access
- NEVER cache entitlements for more than 15 minutes
- NEVER expose customer entitlement details to other customers
- NEVER modify entitlement enforcement without testing

## Phase 3 — Rollback

1. If entitlement enforcement was disabled for debugging, re-enable immediately
2. If entitlement cache was cleared, allow it to repopulate naturally
3. If application access was granted manually, revoke and restore entitlement-based access
4. Verify all customers have correct access levels after changes
5. Document the entitlement issue and resolution

## Common Issues

- symptoms: "get-entitlements returns empty list for active subscriber"
  diagnosis: "Product code mismatch or entitlement not yet provisioned."
  resolution: "Verify product code. New subscriptions may take a few minutes for entitlements to propagate."

## Output Format

```yaml
root_cause: "entitlement_error — <specific_cause>"
evidence:
  - type: entitlement_failure
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix entitlement verification"
  long_term: "Implement robust entitlement caching and monitoring"
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
