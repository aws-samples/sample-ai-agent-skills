---
title: "F1 — Private Marketplace Configuration"
description: "Diagnose Private Marketplace setup and configuration issues"
status: active
severity: MEDIUM
triggers: ["private marketplace", "approved products", "procurement restriction", "marketplace governance"]
owner: devops-agent
objective: "Resolve Private Marketplace configuration issues"
context: "Private Marketplace is an organization-level feature that restricts which products member accounts can subscribe to. Requires AWS Organizations."
---

## Phase 1 — Triage

MUST:
- Check Private Marketplace status: `aws marketplace-catalog list-entities --catalog AWSMarketplace --entity-type Experience`
- Verify Organizations integration: `aws organizations list-aws-service-access-for-organization --query 'EnabledServicePrincipals[?ServicePrincipal==`aws-marketplace.amazonaws.com`]'`
- Check if the caller has admin permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names aws-marketplace-catalog:StartChangeSet`

SHOULD:
- List approved products in the Private Marketplace experience
- Check for pending change sets affecting the experience

MAY:
- Review organization SCPs that may interact with Private Marketplace
- Check if multiple experiences exist (for different OUs)

## Phase 2 — Remediate

MUST:
- Enable AWS Organizations service access for Marketplace if not enabled
- Create or update Private Marketplace experience from management account or delegated admin
- Add required products to the approved list

SHOULD:
- Configure experience for specific OUs if needed
- Set up approval workflows for product addition requests

MAY:
- Create multiple experiences for different business units
- Automate product approval via Catalog API

## Safety Ratings

```
safety_ratings:
  - "marketplace-catalog list-entities: GREEN — read-only listing"
  - "organizations list-aws-service-access: GREEN — read-only check"
  - "marketplace-catalog start-change-set (Experience): YELLOW — modifies procurement governance"
  - "organizations enable-aws-service-access: YELLOW — enables org-level integration"
```

## Escalation Conditions

- Private Marketplace blocks critical product procurement
- Experience configuration changes affect multiple OUs unexpectedly
- Delegated admin setup requires management account intervention
- Conflicting experiences cause inconsistent procurement policies

## Data Sensitivity

- `aws marketplace-catalog list-entities`: LOW — returns experience metadata
- Approved product list: LOW — contains product identifiers
- Organization structure: MEDIUM — reveals OU hierarchy
- Procurement policies: MEDIUM — reveals governance decisions

## Prohibited Actions

- NEVER disable Private Marketplace without organization admin approval
- NEVER add products to the approved list without procurement review
- NEVER modify experiences affecting other OUs without coordination
- NEVER bypass Private Marketplace restrictions using management account

## Phase 3 — Rollback

1. Revert experience changes by submitting a corrective change set
2. If Private Marketplace was disabled, re-enable: submit EnableExperience change set
3. Remove incorrectly approved products from the experience
4. Verify member accounts see the correct approved product list
5. Notify affected teams of any procurement policy changes

## Common Issues

- symptoms: "Member accounts cannot subscribe to any products"
  diagnosis: "Private Marketplace enabled but no products added to approved list."
  resolution: "Add required products to the Private Marketplace experience via Catalog API."

## Output Format

```yaml
root_cause: "private_marketplace — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Private Marketplace configuration"
  long_term: "Establish product approval workflows"
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
