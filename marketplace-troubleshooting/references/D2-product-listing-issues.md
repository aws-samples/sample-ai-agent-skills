---
title: "D2 — Product Listing Issues"
description: "Diagnose product listing and publishing issues"
status: active
severity: MEDIUM
triggers: ["product listing failed", "cannot publish product", "listing error", "product submission"]
owner: devops-agent
objective: "Resolve product listing and publishing issues on Marketplace"
context: "Product listings are managed via the Marketplace Catalog API. Changes go through asynchronous review. Listings require complete metadata, pricing, and fulfillment configuration."
---

## Phase 1 — Triage

MUST:
- Check entity status: `aws marketplace-catalog describe-entity --catalog AWSMarketplace --entity-id <entity-id>`
- List recent change sets: `aws marketplace-catalog list-change-sets --catalog AWSMarketplace --filter-list '[{"Name":"EntityId","ValueList":["<entity-id>"]}]'`
- Check for failed change sets: `aws marketplace-catalog describe-change-set --catalog AWSMarketplace --change-set-id <changeset-id>`

SHOULD:
- Verify all required listing fields are populated
- Check AMI/container image validation status

MAY:
- Review listing against Marketplace content guidelines
- Check seller account standing

## Phase 2 — Remediate

MUST:
- Fix validation errors reported in failed change sets
- Ensure all required metadata fields are populated (title, description, pricing, support)
- For AMI products, ensure AMI passes security scanning

SHOULD:
- Submit corrected change set: `aws marketplace-catalog start-change-set --catalog AWSMarketplace --change-set '[{"ChangeType":"UpdateProduct","Entity":{"Type":"AmiProduct@1.0","Identifier":"<id>"},"Details":"<json>"}]'`
- Test product in limited release before public listing

MAY:
- Set up automated listing updates via CI/CD
- Configure product usage instructions and documentation

## Safety Ratings

```
safety_ratings:
  - "marketplace-catalog describe-entity: GREEN — read-only entity check"
  - "marketplace-catalog list-change-sets: GREEN — read-only listing"
  - "marketplace-catalog describe-change-set: GREEN — read-only status check"
  - "marketplace-catalog start-change-set: YELLOW — modifies product listing"
```

## Escalation Conditions

- Product listing rejected by AWS review with unclear reasons
- AMI security scan fails with false positives
- Pricing configuration rejected by validation
- Product requires restricted category approval

## Data Sensitivity

- `aws marketplace-catalog describe-entity`: MEDIUM — reveals product configuration
- Change set details: MEDIUM — contains product metadata and pricing
- AMI contents: HIGH — may contain proprietary software
- Seller financial terms: HIGH — contains pricing and revenue share details

## Prohibited Actions

- NEVER publish products with known security vulnerabilities
- NEVER set pricing without seller approval
- NEVER modify production listings without testing in staging
- NEVER include customer data in product descriptions or metadata

## Phase 3 — Rollback

1. Cancel pending change sets: `aws marketplace-catalog cancel-change-set --catalog AWSMarketplace --change-set-id <id>`
2. Revert product to previous version by submitting a new change set with old configuration
3. If product was published in error, submit a change set to restrict visibility
4. Remove test AMIs or container images from the listing
5. Document the listing issue and corrective actions

## Common Issues

- symptoms: "Change set fails with VALIDATION_ERROR"
  diagnosis: "Required fields missing or invalid values in product metadata."
  resolution: "Review change set error details. Fix missing fields and resubmit."

## Output Format

```yaml
root_cause: "product_listing — <specific_cause>"
evidence:
  - type: listing_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix listing validation errors"
  long_term: "Automate listing updates via CI/CD"
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
