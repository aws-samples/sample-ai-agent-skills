---
title: "A2 — Procurement Errors"
description: "Diagnose AWS Marketplace procurement workflow errors"
status: active
severity: HIGH
triggers: ["procurement failed", "cannot procure", "purchase error", "contract error"]
owner: devops-agent
objective: "Resolve procurement workflow errors for Marketplace products"
context: "Procurement involves contract acceptance, pricing agreement, and payment processing. Errors can occur at any stage."
---

## Phase 1 — Triage

MUST:
- Check procurement permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names aws-marketplace:Subscribe aws-marketplace:CreateAgreement`
- Verify account standing: check Billing console for account status
- Review recent procurement attempts: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=aws-marketplace.amazonaws.com --max-results 20`

SHOULD:
- Check for pending agreements: `aws marketplace-agreement search-agreements --catalog AWSMarketplace --filters '[{"name":"Status","values":["ACTIVE","PENDING"]}]'`
- Verify Private Marketplace allows the product

MAY:
- Check if the product requires a private offer
- Review organization procurement policies

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for procurement actions
- Resolve account standing issues through Billing console
- Accept pending agreements if appropriate

SHOULD:
- Configure procurement approval workflows
- Set up budget alerts for Marketplace spending: `aws budgets create-budget --account-id <account-id> --budget file://marketplace-budget.json`

MAY:
- Negotiate private offers for volume discounts
- Set up consolidated billing for organization procurement

## Safety Ratings

```
safety_ratings:
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "marketplace-agreement search-agreements: GREEN — read-only agreement listing"
  - "budgets create-budget: YELLOW — creates billing alert, non-destructive"
```

## Escalation Conditions

- Procurement fails with contract validation errors after permission fixes
- Enterprise contract requires legal review
- Procurement blocked by organization-level policies
- Payment processing failures requiring AWS Support intervention

## Data Sensitivity

- `aws cloudtrail lookup-events`: MEDIUM — may reveal procurement details and user identities
- `aws marketplace-agreement search-agreements`: MEDIUM — contains contract terms and pricing
- Budget configurations: LOW — contains threshold values only
- Contract details: HIGH — may contain negotiated pricing terms

## Prohibited Actions

- NEVER accept contracts or agreements without explicit user approval
- NEVER share negotiated pricing terms outside authorized personnel
- NEVER modify organization procurement policies without admin approval
- NEVER bypass procurement approval workflows

## Phase 3 — Rollback

1. If an agreement was accepted in error, contact AWS Support for cancellation options
2. If budget alerts were misconfigured, update or delete: `aws budgets delete-budget --account-id <id> --budget-name <name>`
3. If IAM permissions were over-provisioned, scope down to least privilege
4. Document the procurement error and notify finance team
5. Verify no unexpected charges from partial procurement

## Common Issues

- symptoms: "CreateAgreement returns ValidationException"
  diagnosis: "Agreement terms do not match the offer or product configuration."
  resolution: "Verify offer terms match expected configuration. Check for expired offers."

## Output Format

```yaml
root_cause: "procurement_error — <specific_cause>"
evidence:
  - type: procurement_failure
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix procurement blocker"
  long_term: "Establish procurement approval workflows"
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
