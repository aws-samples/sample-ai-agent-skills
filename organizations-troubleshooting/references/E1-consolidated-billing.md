---
title: "E1 — Consolidated Billing Issues"
description: "Diagnose consolidated billing problems and cost visibility"
status: active
severity: MEDIUM
triggers:
  - "consolidated billing"
  - "billing issue"
  - "cost visibility"
  - "payer account"
  - "linked account billing"
owner: devops-agent
objective: "Resolve consolidated billing issues and ensure correct cost attribution"
context: "All member accounts in an organization use consolidated billing through the management account. The management account is the payer. RI and SP benefits are shared by default. Volume discounts are calculated on aggregate usage. Cost allocation tags must be activated at the management account level."
---

## Phase 1 — Triage

MUST:
- Verify organization billing setup: `aws organizations describe-organization`
- Check account billing status: `aws organizations describe-account --account-id <account-id>`
- Verify the management account payment method is valid
- Check Cost Explorer for cost visibility: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity MONTHLY --metrics BlendedCost --group-by Type=DIMENSION,Key=LINKED_ACCOUNT`

SHOULD:
- Check RI/SP sharing settings
- Verify cost allocation tags are activated
- Review billing preferences in the management account

MAY:
- Check for billing anomalies across accounts
- Review the billing dashboard for payment issues

## Phase 2 — Remediate

MUST:
- Ensure the management account has a valid payment method
- Activate required cost allocation tags at the management account level
- Verify cost data is flowing correctly to Cost Explorer

SHOULD:
- Configure RI/SP sharing preferences per account
- Set up billing alerts and budgets
- Enable Cost Explorer for member accounts if needed

MAY:
- Use AWS Cost Anomaly Detection for automated monitoring
- Implement chargeback/showback using cost allocation tags

## Common Issues

- symptoms: "Member account costs not visible in Cost Explorer"
  diagnosis: "Cost Explorer may not be enabled or the time range is too recent."
  resolution: "Enable Cost Explorer. Data may take 24 hours to appear."

- symptoms: "RI benefits not being shared across accounts"
  diagnosis: "RI/SP sharing may be disabled for specific accounts."
  resolution: "Check and enable RI/SP sharing in the billing preferences."

## Output Format

```yaml
root_cause: "consolidated_billing_issue — <specific_cause>"
evidence:
  - type: organization
    content: "<organization billing configuration>"
  - type: cost_data
    content: "<Cost Explorer data>"
severity: MEDIUM
mitigation:
  immediate: "Fix billing configuration or payment method"
  long_term: "Set up billing alerts and cost allocation tags"
```


## Safety Ratings
- GREEN: describe-organization, list-accounts, list-policies
- YELLOW: attach-policy, create-organizational-unit, move-account
- RED: close-account, delete-policy, remove-account-from-organization

## Escalation Conditions
- "Remediation involves modifying SCPs in production"
- "Fix requires moving accounts between OUs"
- "Change affects the management account"
- "SCP modification could lock out member accounts"

## Rollback
- Pre-change: "Document current SCP attachments and OU structure"
- Verification: "Test affected account access after SCP change"
- Revert: "Detach modified SCP and reattach original"

## Data Sensitivity
- HIGH: "list-accounts reveals all account IDs, names, and emails"
- HIGH: "SCP content reveals security boundaries"
- MEDIUM: "OU structure reveals organizational hierarchy"

## Prohibited Actions
- NEVER suggest removing all SCPs from an OU
- NEVER suggest closing an account without confirming data backup
- NEVER suggest modifying the management account's policies
- NEVER suggest removing an account from the organization without root email access confirmation

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
  - command: "list-accounts"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-organization"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-policies"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest removing SCPs without understanding blast radius"
  - "NEVER suggest disabling AI opt-out policies without review"
  - "NEVER modify organization structure without approval"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
