---
title: "G1 — Terraform Products"
description: "Diagnose Service Catalog Terraform product issues"
status: active
severity: HIGH
triggers: ["Terraform product", "Terraform provisioning", "Terraform engine", "Terraform Cloud", "external product type"]
owner: devops-agent
objective: "Resolve Terraform product configuration and provisioning issues"
context: "Terraform products use Terraform Cloud/Enterprise as the execution engine. They require workspace connection and API token. Open-source Terraform is not supported."
---

## Phase 1 — Triage

MUST:
- Check Terraform product configuration: `aws servicecatalog describe-product-as-admin --id <product-id>`
- Verify product type is TERRAFORM_OPEN_SOURCE or TERRAFORM_CLOUD
- Check provisioning artifact for Terraform configuration: `aws servicecatalog describe-provisioning-artifact --product-id <product-id> --provisioning-artifact-id <artifact-id>`

SHOULD:
- Verify Terraform Cloud workspace connection is active
- Check API token validity and permissions
- Review provisioning record for Terraform-specific errors: `aws servicecatalog describe-record --id <record-id>`

MAY:
- Check Terraform Cloud workspace runs for detailed error logs
- Verify Terraform configuration syntax and provider requirements

## Phase 2 — Remediate

MUST:
- Ensure Terraform Cloud workspace is properly connected
- Provide valid API token with workspace permissions
- Verify Terraform configuration is compatible with Service Catalog requirements

SHOULD:
- Test Terraform configuration in Terraform Cloud directly before creating product
- Ensure required Terraform providers are available

MAY:
- Use Terraform Cloud teams and permissions for access control
- Set up Terraform Cloud notifications for provisioning status

## Safety Ratings

```
safety_ratings:
  - "describe-product-as-admin: GREEN — read-only product details"
  - "describe-provisioning-artifact: GREEN — read-only artifact details"
  - "provision-product (Terraform): YELLOW — creates resources via Terraform Cloud"
  - "terminate-provisioned-product (Terraform): RED — destroys Terraform-managed resources"
```

## Escalation Conditions

- Terraform Cloud workspace connection requires Terraform Cloud admin access
- API token expired and renewal requires Terraform Cloud organization admin
- Terraform provider issues require provider-specific troubleshooting
- Terraform state corruption requires Terraform Cloud state management

## Data Sensitivity

- `aws servicecatalog describe-product-as-admin`: LOW — product metadata
- Terraform Cloud API token: HIGH — provides access to Terraform Cloud workspace
- Terraform state: HIGH — contains resource details and potentially sensitive outputs
- Terraform configuration: MEDIUM — contains infrastructure definitions

## Prohibited Actions

- NEVER store Terraform Cloud API tokens in plain text
- NEVER modify Terraform state directly without understanding consequences
- NEVER use open-source Terraform expecting it to work with Service Catalog
- NEVER provision Terraform products without testing the configuration first

## Phase 3 — Rollback

1. If Terraform provisioning created partial resources: terminate provisioned product to trigger Terraform destroy
2. If workspace connection was broken: reconnect and verify state consistency
3. If API token was rotated: update Service Catalog configuration with new token
4. If Terraform state is corrupted: restore from Terraform Cloud state history
5. Document Terraform product issues and update configuration

## Common Issues

- symptoms: "Terraform product provisioning fails with 'Failed to connect to Terraform Cloud workspace'"
  diagnosis: "Terraform Cloud workspace connection is not configured or API token is invalid."
  resolution: "Verify Terraform Cloud workspace exists and is accessible. Update API token if expired. Ensure workspace allows Service Catalog integration."

## Output Format

```yaml
root_cause: "terraform_product — <specific_cause>"
evidence:
  - type: terraform_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix Terraform product configuration"
  long_term: "Implement Terraform Cloud workspace management automation"
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
