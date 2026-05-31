---
title: "D3 — Third-Party Integration"
description: "Diagnose Security Hub third-party product integration issues"
status: active
severity: MEDIUM
triggers:
  - "third-party integration"
  - "partner integration"
  - "product integration"
  - "external findings"
  - "marketplace integration"
owner: devops-agent
objective: "Identify and resolve third-party product integration issues with Security Hub"
context: "Security Hub integrates with third-party security products that send findings in ASFF format. Issues arise from product registration, ASFF format compliance, IAM permissions, or product-specific configuration. Third-party products must be enabled as finding providers."
---

## Phase 1 — Triage

MUST:
- List available products: `aws securityhub describe-products`
- List enabled product integrations: `aws securityhub list-enabled-products-for-import`
- Check if the third-party product is enabled
- Verify findings from the product: `aws securityhub get-findings --filters '{"ProductArn":[{"Value":"<product-arn>","Comparison":"EQUALS"}]}' --max-results 10`
- Check CloudTrail for BatchImportFindings errors from the product

SHOULD:
- Verify the product's IAM role has securityhub:BatchImportFindings permission
- Check the product's configuration for Security Hub integration settings
- Verify the product is sending findings in valid ASFF format

MAY:
- Check the product's documentation for Security Hub integration requirements
- Verify network connectivity between the product and Security Hub API

## Phase 2 — Remediate

MUST:
- Enable the product integration if not enabled: `aws securityhub enable-import-findings-for-product --product-arn <product-arn>`
- Fix IAM permissions for the product's finding import role
- Verify findings start appearing after enabling

SHOULD:
- Configure the product to send findings to Security Hub
- Validate ASFF format compliance for the product's findings

MAY:
- Set up monitoring for finding import from the product
- Create automation rules for third-party findings

## Common Issues

- symptoms: "Third-party product enabled but no findings appearing"
  diagnosis: "The product is not configured to send findings, or the IAM role lacks permissions."
  resolution: "Configure the product's Security Hub integration. Verify the IAM role has securityhub:BatchImportFindings."

- symptoms: "BatchImportFindings failing for third-party product"
  diagnosis: "The findings do not conform to ASFF format or the ProductArn is incorrect."
  resolution: "Validate ASFF format. Ensure the ProductArn matches the registered product ARN."

- symptoms: "Product integration was working but stopped"
  diagnosis: "The product's IAM credentials expired, the integration was disabled, or the product configuration changed."
  resolution: "Check product configuration, IAM credentials, and integration status."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable product integration | YELLOW | Configuration change; reversible |
| Fix IAM permissions | YELLOW | Permission change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Third-party product integration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-products` | LOW | Product catalog |
| `list-enabled-products-for-import` | LOW | Integration list |
| `get-findings` (product) | HIGH | Third-party security findings |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Product integration enablement | Disable via `disable-import-findings-for-product` |
| IAM permission change | Revert IAM policy |

## Output Format

```yaml
root_cause: "third_party_integration — <specific_cause>"
evidence:
  - type: product_status
    content: "<product integration status>"
  - type: findings
    content: "<findings from the product>"
  - type: import_errors
    content: "<BatchImportFindings errors>"
severity: MEDIUM
mitigation:
  immediate: "Enable the integration and fix configuration issues"
  long_term: "Monitor finding import and maintain product configuration"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
