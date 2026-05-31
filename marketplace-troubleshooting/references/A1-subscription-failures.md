---
title: "A1 — Subscription Failures"
description: "Diagnose AWS Marketplace subscription failures"
status: active
severity: HIGH
triggers: ["cannot subscribe", "subscription failed", "subscribe error", "marketplace subscription"]
owner: devops-agent
objective: "Resolve Marketplace subscription failures and restore procurement capability"
context: "Marketplace subscriptions are per-account billing agreements. Failures can stem from IAM permissions, payment method issues, or Private Marketplace restrictions."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for aws-marketplace:Subscribe: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names aws-marketplace:Subscribe`
- Verify payment method is valid: check AWS Billing console for payment method status
- Check if Private Marketplace is restricting the product: `aws marketplace-catalog list-entities --catalog AWSMarketplace --entity-type Experience`

SHOULD:
- Review CloudTrail for Subscribe events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Subscribe --max-results 10`
- Check for SCPs blocking marketplace actions: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`

MAY:
- Verify the product is still available on Marketplace
- Check if the product requires a specific agreement type (contract vs subscription)

## Phase 2 — Remediate

MUST:
- Fix IAM permissions — ensure aws-marketplace:Subscribe and aws-marketplace:ViewSubscriptions are granted
- Resolve payment method issues through Billing console
- If Private Marketplace blocks the product, request addition to the approved list

SHOULD:
- Add the product to Private Marketplace approved list if appropriate: `aws marketplace-catalog start-change-set --catalog AWSMarketplace --change-set '[{"ChangeType":"AllowProductProcurement","Entity":{"Type":"Experience@1.0","Identifier":"<experience-id>"},"Details":"{\"ProductId\":\"<product-id>\"}"}]'`
- Verify SCP allows marketplace actions

MAY:
- Set up procurement approval workflows for future subscriptions
- Configure SNS notifications for subscription events

## Safety Ratings

```
safety_ratings:
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "marketplace-catalog list-entities: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "marketplace-catalog start-change-set: YELLOW — modifies Private Marketplace configuration"
```

## Escalation Conditions

- Subscription fails with internal service errors after IAM and payment fixes
- Private Marketplace changes require organization admin approval
- SCP modifications needed that affect other accounts
- Subscription requires enterprise contract negotiation

## Data Sensitivity

- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — may reveal user identities and actions
- `aws marketplace-catalog list-entities`: LOW — returns entity metadata
- Payment method details: HIGH — contains financial information

## Prohibited Actions

- NEVER share payment method details in logs or outputs
- NEVER subscribe to products without explicit user approval and cost acknowledgment
- NEVER modify Private Marketplace settings without organization admin approval
- NEVER bypass SCPs by switching to management account without authorization

## Phase 3 — Rollback

1. If a subscription was created in error, cancel it: navigate to Marketplace console > Manage subscriptions > Cancel
2. If Private Marketplace was modified incorrectly, revert the change set
3. If IAM permissions were over-provisioned, scope them down to specific product codes
4. Document the subscription attempt and notify the procurement team
5. Verify no unexpected charges were incurred

## Common Issues

- symptoms: "Subscribe returns AccessDeniedException"
  diagnosis: "IAM policy missing aws-marketplace:Subscribe permission or SCP blocking."
  resolution: "Add aws-marketplace:Subscribe to IAM policy. Check SCPs with aws organizations list-policies."

- symptoms: "Subscribe fails with payment method error"
  diagnosis: "No valid payment method on the account or payment method expired."
  resolution: "Update payment method in Billing console. Verify account is not suspended."

## Output Format

```yaml
root_cause: "subscription_failure — <specific_cause>"
evidence:
  - type: subscription_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix subscription blocker"
  long_term: "Set up procurement workflows and Private Marketplace"
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
