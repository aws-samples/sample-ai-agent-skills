---
title: "Z1 — General Troubleshooting"
description: "General Service Catalog troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Service Catalog error", "Service Catalog issue", "Service Catalog not working"]
owner: devops-agent
objective: "Diagnose and resolve general Service Catalog issues using systematic investigation"
context: "Catch-all runbook for Service Catalog issues. Start here when the specific problem category is unclear."
---

## Phase 1 — Triage

MUST:
- Identify the symptom category: portfolio, product, constraint, provisioned product, governance, or sharing
- Check portfolios: `aws servicecatalog list-portfolios`
- Check provisioned products: `aws servicecatalog scan-provisioned-products --query 'ProvisionedProducts[?Status!=`AVAILABLE`]'`

SHOULD:
- Review CloudTrail for Service Catalog events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=servicecatalog.amazonaws.com --max-results 20`
- Check IAM permissions for the current user/role

MAY:
- Check AWS Health Dashboard for Service Catalog service issues
- Compare behavior across different accounts or regions

## Phase 2 — Remediate

MUST:
- Route to specific runbook once symptom category is identified:
  - Portfolio issues → A1-portfolio-creation or A2-portfolio-sharing
  - Product provisioning → B1-product-provisioning-failures or B2-product-version-issues
  - Constraints → C1-launch-constraint-errors or C2-template-constraint-issues
  - Provisioned products → D1-provisioned-product-failures or D2-update-failures
  - Governance → E1-tag-option-issues or E2-budget-integration
  - Organization → F1-organization-sharing or F2-stackset-constraints
  - Terraform → G1-terraform-products

SHOULD:
- Document the issue for pattern recognition
- Check if issue is reproducible

MAY:
- Open AWS Support case if issue persists after runbook remediation
- Check AWS forums and re:Post for similar reported issues

## Safety Ratings

```
safety_ratings:
  - "list-portfolios / scan-provisioned-products: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only event history"
  - "describe-* APIs: GREEN — read-only inspection"
  - "terminate-provisioned-product: RED — deletes resources"
```

## Escalation Conditions

- Issue persists after following specific runbook remediation steps
- Multiple users or accounts affected simultaneously
- Issue involves AWS service degradation or outage
- Security-related issue requiring immediate attention

## Data Sensitivity

- `aws servicecatalog list-portfolios`: LOW — portfolio metadata
- `aws servicecatalog scan-provisioned-products`: LOW — product status
- `aws cloudtrail lookup-events`: MEDIUM — user activity records
- Provisioned product details: MEDIUM — may contain resource configurations

## Prohibited Actions

- NEVER terminate provisioned products without confirming data backup
- NEVER escalate to AWS Support without first following relevant runbook
- NEVER assume the issue is Service Catalog-specific without checking IAM and CloudFormation
- NEVER share provisioned product details containing sensitive configurations

## Phase 3 — Rollback

1. If any remediation step made the issue worse: revert the specific change
2. If provisioned product was terminated: re-provision and restore data from backup
3. If IAM changes were made: revert to original policy configuration
4. If constraints were modified: restore original constraint configuration
5. Document all changes made during troubleshooting

## Common Issues

- symptoms: "End user cannot see any products in Service Catalog console"
  diagnosis: "User is not associated with any portfolio, or user lacks servicecatalog:SearchProducts permission."
  resolution: "Associate user/role with a portfolio: aws servicecatalog associate-principal-with-portfolio. Grant ServiceCatalog end user permissions."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Apply specific runbook remediation"
  long_term: "Document issue pattern and implement preventive measures"
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
