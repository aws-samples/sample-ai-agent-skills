---
title: "F1 — Linked Account Costs"
description: "Diagnose linked account cost visibility issues in Organizations"
status: active
severity: MEDIUM
triggers:
  - "cannot see linked account costs"
  - "member account no cost data"
  - "organizational cost visibility"
  - "account cost access"
owner: devops-agent
objective: "Resolve linked account cost visibility and access issues"
context: "In AWS Organizations, Cost Explorer access for member accounts must be enabled by the management account. By default, member accounts cannot see cost data. The management account can see all accounts' costs."
---

## Phase 1 — Triage

MUST:
- Check if Cost Explorer is enabled for the organization: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity MONTHLY --metrics UnblendedCost` (from management account)
- Verify member account access is enabled (requires management account console)
- Check if the user has ce:* IAM permissions
- Verify the account is a member of the organization: `aws organizations list-accounts`

SHOULD:
- Check IAM policies for Cost Explorer access restrictions
- Verify no SCP is blocking Cost Explorer access
- Check if the member account was recently added

MAY:
- Review organization structure: `aws organizations list-organizational-units-for-parent --parent-id <root-id>`
- Check if cost allocation tags are shared across the organization

## Phase 2 — Remediate

MUST:
- Enable Cost Explorer access for member accounts (management account action)
- Grant IAM permissions for ce:* actions to member account users
- Ensure no SCP blocks Cost Explorer access

SHOULD:
- Set up cross-account cost visibility policies
- Enable cost allocation tags at the organization level
- Configure CUR with linked account breakdown

MAY:
- Implement cost dashboards accessible to member accounts
- Set up per-account budgets from the management account

## Common Issues

- symptoms: "Member account sees no cost data"
  diagnosis: "Cost Explorer access not enabled for member accounts."
  resolution: "Management account must enable member account access in Billing preferences."

- symptoms: "Can see own costs but not other accounts"
  diagnosis: "Expected behavior for member accounts. Only management account sees all."
  resolution: "Use management account for cross-account analysis. Share reports via CUR."

- symptoms: "New member account has no historical costs"
  diagnosis: "Cost data only available from when the account joined the organization."
  resolution: "Historical data before joining is not available in organizational view."

## Output Format

```yaml
root_cause: "linked_account_costs — <specific_cause>"
evidence:
  - type: org_config
    content: "<organization and member account status>"
  - type: access_config
    content: "<Cost Explorer access settings>"
severity: MEDIUM
mitigation:
  immediate: "Enable member account access and fix IAM permissions"
  long_term: "Implement organizational cost visibility strategy"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Cross-account cost data reveals organizational spending |
| Blast Radius | LOW — Cost visibility changes affect reporting only |
| Rollback Complexity | LOW — Access settings can be toggled |
| Compliance Scope | MEDIUM — Cross-account cost visibility supports financial governance |

## Escalation Conditions

- Escalate to management account owner if member account access needs to be enabled
- Escalate to security if SCPs are blocking legitimate cost access
- Escalate to finance if cost visibility gaps affect organizational reporting

## Data Sensitivity

- **Cross-account cost data** reveals per-account spending — restrict to authorized personnel
- **Organization structure** is exposed through cost breakdowns
- **Member account IDs** are visible in cost reports

## PROHIBITED Actions

- Do NOT enable member account cost access without management account owner approval
- Do NOT share cross-account cost data outside authorized finance teams
- Do NOT modify SCPs affecting cost access without understanding the impact on all accounts

## Phase 3 — Rollback

- Disable member account access if it was enabled incorrectly
- Restore SCP to previous state if it was modified
- Revert IAM permission changes made during troubleshooting

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
