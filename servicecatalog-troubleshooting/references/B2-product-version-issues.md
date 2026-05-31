---
title: "B2 — Product Version Issues"
description: "Diagnose Service Catalog product version and provisioning artifact issues"
status: active
severity: MEDIUM
triggers: ["product version", "provisioning artifact", "template update", "version not available"]
owner: devops-agent
objective: "Resolve product version management and provisioning artifact issues"
context: "Product versions (provisioning artifacts) are immutable. New versions must be created for template changes. Versions can be active or inactive."
---

## Phase 1 — Triage

MUST:
- List product versions: `aws servicecatalog list-provisioning-artifacts --product-id <product-id>`
- Check specific version details: `aws servicecatalog describe-provisioning-artifact --product-id <product-id> --provisioning-artifact-id <artifact-id>`
- Verify version is active and not disabled

SHOULD:
- Check if the template URL is accessible: verify S3 bucket permissions
- Validate the CloudFormation template: `aws cloudformation validate-template --template-url <url>`

MAY:
- Compare versions to identify changes: review template differences
- Check if provisioned products reference the problematic version

## Phase 2 — Remediate

MUST:
- Create new version for template changes: `aws servicecatalog create-provisioning-artifact --product-id <product-id> --parameters Name=v2,Info={LoadTemplateFromURL=<url>},Type=CLOUD_FORMATION_TEMPLATE`
- Deactivate broken versions: `aws servicecatalog update-provisioning-artifact --product-id <product-id> --provisioning-artifact-id <artifact-id> --active`
- Ensure template is accessible from the S3 URL

SHOULD:
- Set guidance on old versions to DEPRECATED to warn users
- Update provisioned products to use the new version

MAY:
- Automate version creation with CI/CD pipeline
- Use version descriptions to document changes

## Safety Ratings

```
safety_ratings:
  - "list-provisioning-artifacts: GREEN — read-only version listing"
  - "create-provisioning-artifact: YELLOW — creates new product version"
  - "update-provisioning-artifact (deactivate): YELLOW — hides version from users"
  - "delete-provisioning-artifact: RED — removes version permanently"
```

## Escalation Conditions

- Template S3 bucket is in a different account and access is denied
- Provisioned products cannot be updated to new version due to breaking changes
- Version creation fails with template validation errors
- Multiple products affected by the same template issue

## Data Sensitivity

- `aws servicecatalog list-provisioning-artifacts`: LOW — version metadata
- `aws servicecatalog describe-provisioning-artifact`: MEDIUM — may contain template URL
- `aws cloudformation validate-template`: LOW — template structure validation
- Template content: MEDIUM — may contain resource configurations

## Prohibited Actions

- NEVER delete product versions that are referenced by active provisioned products
- NEVER modify S3 template files expecting existing versions to update (versions are immutable)
- NEVER create versions with untested templates in production portfolios
- NEVER deactivate all versions of a product without a replacement

## Phase 3 — Rollback

1. If wrong version was created: deactivate it and set guidance to DEPRECATED
2. If version was accidentally deactivated: reactivate with `--active` flag
3. If provisioned products were updated to broken version: update back to previous version
4. If template S3 object was deleted: restore from S3 versioning or re-upload
5. Document version changes and notify portfolio consumers

## Common Issues

- symptoms: "End user cannot see the latest product version when provisioning"
  diagnosis: "New version was created but not set as active, or user is viewing a cached product list."
  resolution: "Verify version is active: aws servicecatalog describe-provisioning-artifact. Refresh the Service Catalog console."

## Output Format

```yaml
root_cause: "product_version — <specific_cause>"
evidence:
  - type: version_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix version availability"
  long_term: "Implement version management workflow with CI/CD"
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
