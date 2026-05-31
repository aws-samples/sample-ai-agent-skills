---
title: "B1 — Subscription Errors"
description: "Diagnose Data Exchange subscription failures"
status: active
severity: HIGH
triggers: ["subscription failed", "cannot subscribe", "subscription pending", "subscription rejected", "Marketplace subscription error"]
owner: devops-agent
objective: "Resolve subscription issues including Marketplace integration, approval workflows, and billing"
context: "Subscriptions are managed through AWS Marketplace. Some products require provider approval. Subscriptions can be free, one-time, or recurring."
---

## Phase 1 — Triage

MUST:
- Check subscription status: `aws dataexchange list-data-sets --origin ENTITLED --query 'DataSets[*].{Name:Name,Id:Id,Origin:Origin}'`
- Review Marketplace subscriptions: `aws marketplace-catalog list-entities --catalog AWSMarketplace --entity-type Offer`
- Check CloudTrail for subscription events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=dataexchange.amazonaws.com --max-results 20`

SHOULD:
- Verify AWS Marketplace account is in good standing
- Check if the product requires provider approval
- Verify billing information is valid

MAY:
- Contact the data provider for approval status
- Review Marketplace billing dashboard for payment issues

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for Marketplace subscription operations
- Resolve billing issues preventing subscription activation
- Accept pending subscription terms if required

SHOULD:
- Verify the product is available in the target region
- Check that the subscription offer has not expired

MAY:
- Contact provider for expedited approval
- Set up subscription auto-renewal if applicable

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-data-sets --origin ENTITLED: GREEN — read-only listing"
  - "marketplace-catalog list-entities: GREEN — read-only Marketplace query"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "Marketplace subscribe: YELLOW — creates billing commitment"
```

## Escalation Conditions

- Subscription rejected by provider without explanation
- Billing issues prevent subscription activation and cannot be resolved
- Marketplace integration errors require AWS support
- Subscription stuck in PENDING for more than 48 hours

## Data Sensitivity

- `aws dataexchange list-data-sets --origin ENTITLED`: LOW — returns entitled data set metadata
- `aws marketplace-catalog list-entities`: MEDIUM — reveals Marketplace account details
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history
- `billing information`: HIGH — contains payment and account details

## Prohibited Actions

- NEVER subscribe to paid products without budget approval
- NEVER share Marketplace account credentials to debug subscription issues
- NEVER cancel active subscriptions without verifying data export is complete
- NEVER accept subscription terms without reviewing pricing and data usage agreements

## Phase 3 — Rollback

1. If subscribed to wrong product, cancel subscription through Marketplace
2. Verify no data was exported before cancellation to avoid unnecessary charges
3. If billing was affected, review Marketplace billing for refund eligibility
4. Document the subscription issue and correct product details
5. Re-subscribe to the correct product if needed

## Common Issues

- symptoms: "Subscription stuck in PENDING — awaiting provider approval"
  diagnosis: "Product requires manual provider approval."
  resolution: "Contact the data provider. Check provider's approval SLA."

## Output Format

```yaml
root_cause: "subscription — <specific_cause>"
evidence:
  - type: subscription_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix subscription issue"
  long_term: "Document subscription requirements and approval workflows"
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
