---
title: "G1 — Vendor Insights"
description: "Diagnose Vendor Insights security profile and assessment issues"
status: active
severity: LOW
triggers: ["vendor insights", "security profile", "vendor assessment", "compliance data"]
owner: devops-agent
objective: "Resolve Vendor Insights access and data issues"
context: "Vendor Insights provides security and compliance profiles for Marketplace products. Requires seller opt-in and assessment data submission."
---

## Phase 1 — Triage

MUST:
- Check if Vendor Insights is available for the product: review product detail page in Marketplace console
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names aws-marketplace:GetBuyerDashboard`
- Check subscription status for the product

SHOULD:
- Verify the seller has opted into Vendor Insights
- Check if security profile data is current

MAY:
- Review available compliance certifications
- Check if the product has third-party audit reports

## Phase 2 — Remediate

MUST:
- Ensure IAM permissions include Vendor Insights access
- Verify active subscription to the product (required for detailed profiles)
- Contact seller if security profile data is missing or outdated

SHOULD:
- Subscribe to profile update notifications
- Document compliance requirements for procurement decisions

MAY:
- Integrate Vendor Insights data into procurement workflows
- Set up periodic reviews of vendor security profiles

## Safety Ratings

```
safety_ratings:
  - "Marketplace console Vendor Insights: GREEN — read-only security profile view"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "Vendor Insights API: GREEN — read-only compliance data access"
```

## Escalation Conditions

- Vendor Insights shows critical security findings for a subscribed product
- Seller has not updated security profile in over 12 months
- Compliance certifications have expired
- Security profile data contradicts seller claims

## Data Sensitivity

- Vendor security profiles: MEDIUM — contains vendor compliance posture
- Compliance certifications: LOW — generally public information
- Assessment reports: MEDIUM — may contain detailed security findings
- Procurement decisions: MEDIUM — reveals organizational risk tolerance

## Prohibited Actions

- NEVER share vendor security profiles outside authorized procurement personnel
- NEVER make procurement decisions solely based on Vendor Insights without additional due diligence
- NEVER assume missing Vendor Insights means the product is insecure
- NEVER modify vendor assessment data

## Phase 3 — Rollback

1. If procurement decisions were made on outdated profiles, reassess with current data
2. If Vendor Insights access was granted too broadly, scope down IAM permissions
3. Document any compliance gaps identified during investigation
4. Notify procurement team of any changes to vendor security posture
5. Update internal vendor risk assessments

## Common Issues

- symptoms: "Vendor Insights not available for a product"
  diagnosis: "Seller has not opted into Vendor Insights program."
  resolution: "Contact seller to request Vendor Insights enrollment. Use alternative compliance assessment methods."

## Output Format

```yaml
root_cause: "vendor_insights — <specific_cause>"
evidence:
  - type: insights_issue
    content: "<error details>"
severity: LOW
mitigation:
  immediate: "Address Vendor Insights access issue"
  long_term: "Integrate Vendor Insights into procurement workflow"
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
