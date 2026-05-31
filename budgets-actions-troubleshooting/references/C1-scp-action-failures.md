---
title: "C1 — SCP Action Failures"
description: "Diagnose Service Control Policy budget action failures"
status: active
severity: CRITICAL
triggers: ["scp action failed", "scp budget action", "service control policy action", "org policy action"]
owner: devops-agent
objective: "Resolve SCP-based budget action failures"
context: "SCP actions attach pre-existing SCPs to OUs or accounts when budget thresholds are breached. Requires AWS Organizations and management account access."
---

## Phase 1 — Triage

MUST:
- Check action configuration: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify SCP exists: `aws organizations describe-policy --policy-id <policy-id>`
- Check Organizations access: `aws organizations describe-organization`

SHOULD:
- Verify execution role has organizations:AttachPolicy permission
- Check if the SCP is already attached to the target
- Review action history: `aws budgets describe-budget-action-histories --account-id <account-id> --budget-name <name> --action-id <action-id>`

MAY:
- Check for conflicting SCPs on the target OU/account
- Verify the SCP content is correct

## Phase 2 — Remediate

MUST:
- Ensure the SCP exists and is in ENABLED state
- Grant organizations:AttachPolicy to the execution role
- Verify the action is configured from the management account or delegated admin

SHOULD:
- Test SCP attachment manually in a sandbox OU first
- Set up approval workflow for SCP actions (highly recommended)

MAY:
- Create a dedicated SCP for budget actions (e.g., deny ec2:RunInstances)
- Implement automated SCP detachment after budget period

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget-action: GREEN — read-only action check"
  - "organizations describe-policy: GREEN — read-only policy check"
  - "organizations describe-organization: GREEN — read-only org check"
  - "organizations attach-policy: RED — restricts permissions for entire OU/account"
```

## Escalation Conditions

- SCP action blocks critical services across multiple accounts
- SCP attachment affects production OUs
- Management account access required but not available
- SCP conflicts with existing policies causing unexpected denials

## Data Sensitivity

- SCP content: HIGH — reveals organization security governance
- Organization structure: MEDIUM — reveals account hierarchy
- Action configuration: MEDIUM — reveals cost control strategy
- Target OUs/accounts: MEDIUM — reveals organizational structure

## Prohibited Actions

- NEVER attach SCPs to the management account (SCPs do not affect it)
- NEVER create deny-all SCPs without understanding the blast radius
- NEVER test SCP actions on production OUs
- NEVER detach SCPs without understanding which budget action attached them

## Phase 3 — Rollback

1. Detach the SCP from the target: `aws organizations detach-policy --policy-id <policy-id> --target-id <ou-id-or-account-id>`
2. Reset the budget action: `aws budgets execute-budget-action --account-id <id> --budget-name <name> --action-id <id> --execution-type RESET_BUDGET_ACTION`
3. Verify affected accounts can access required services after SCP detachment
4. Check for cascading effects on child OUs
5. Document the SCP action and rollback for governance records

## Common Issues

- symptoms: "SCP action fails with AccessDeniedException"
  diagnosis: "Execution role missing organizations:AttachPolicy or caller is not management account."
  resolution: "Grant organizations:AttachPolicy to execution role. Ensure action is configured from management account."

## Output Format

```yaml
root_cause: "scp_action — <specific_cause>"
evidence:
  - type: scp_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix SCP action configuration"
  long_term: "Implement approval workflows for SCP actions"
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
