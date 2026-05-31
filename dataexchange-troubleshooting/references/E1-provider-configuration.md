---
title: "E1 — Provider Configuration"
description: "Diagnose Data Exchange provider setup and configuration issues"
status: active
severity: MEDIUM
triggers: ["provider registration error", "cannot publish data", "Marketplace listing failed", "provider account setup"]
owner: devops-agent
objective: "Resolve provider configuration issues including Marketplace registration, product listing, and publishing workflows"
context: "Providers must register with AWS Marketplace, configure products, and set up publishing workflows. Provider configuration affects all subscribers."
---

## Phase 1 — Triage

MUST:
- Verify provider registration: `aws marketplace-catalog list-entities --catalog AWSMarketplace --entity-type DataProduct`
- List provider data sets: `aws dataexchange list-data-sets --query 'DataSets[?Origin==`OWNED`]'`
- Check IAM permissions for provider operations

SHOULD:
- Verify Marketplace seller registration is complete
- Check product listing status in Marketplace
- Review publishing workflow configuration

MAY:
- Check CloudTrail for provider API errors
- Verify S3 bucket configuration for data source

## Phase 2 — Remediate

MUST:
- Complete Marketplace seller registration if not done
- Fix IAM permissions for dataexchange:Create*, dataexchange:Update*
- Ensure product listing meets Marketplace requirements

SHOULD:
- Configure automated publishing workflows with event actions
- Set up proper data set descriptions and documentation

MAY:
- Configure subscription verification for controlled access
- Set up CloudWatch monitoring for publishing operations

## Safety Ratings

```
safety_ratings:
  - "marketplace-catalog list-entities: GREEN — read-only listing"
  - "dataexchange list-data-sets: GREEN — read-only listing"
  - "Marketplace product listing: YELLOW — makes data publicly discoverable"
  - "dataexchange create-data-set: YELLOW — creates new data set resource"
```

## Escalation Conditions

- Marketplace seller registration blocked by compliance requirements
- Product listing rejected by Marketplace review
- Provider account suspended or restricted
- Publishing workflow fails consistently with service errors

## Data Sensitivity

- `aws marketplace-catalog list-entities`: MEDIUM — reveals product listing details
- `aws dataexchange list-data-sets`: LOW — returns data set metadata
- `Marketplace seller information`: HIGH — contains business and banking details
- `product pricing configuration`: MEDIUM — contains pricing strategy

## Prohibited Actions

- NEVER publish data sets containing PII without proper data classification
- NEVER set pricing to zero for commercial data without business approval
- NEVER modify product listings during active subscriber usage without notification
- NEVER share Marketplace seller credentials with unauthorized personnel

## Phase 3 — Rollback

1. If a product was listed incorrectly, unpublish from Marketplace
2. If data sets were created with wrong configuration, delete (if no subscribers)
3. Revert IAM policy changes if they were too permissive
4. Verify no unauthorized data was published
5. Document the configuration issue and correct settings

## Common Issues

- symptoms: "Cannot list product on Marketplace — seller registration incomplete"
  diagnosis: "AWS Marketplace seller registration not completed."
  resolution: "Complete seller registration including tax and banking information."

## Output Format

```yaml
root_cause: "provider_configuration — <specific_cause>"
evidence:
  - type: provider_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix provider configuration"
  long_term: "Document provider setup checklist"
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
