---
title: "C1 — Admin Account Issues"
description: "Diagnose Detective delegated administrator account configuration problems"
status: active
severity: HIGH
triggers:
  - "admin account"
  - "delegated administrator"
  - "detective admin"
  - "organization admin"
  - "admin configuration"
owner: devops-agent
objective: "Resolve admin account configuration issues for Detective"
context: "Detective uses a delegated administrator account to manage the behavior graph and member accounts. The admin account is set through Organizations. Changing the admin requires disabling and re-enabling the service, which resets all data."
---

## Phase 1 — Triage

MUST:
- Check current delegated admin: `aws organizations list-delegated-administrators --service-principal detective.amazonaws.com`
- Verify the admin account has Detective enabled: `aws detective list-graphs`
- Check Organizations integration status
- Verify IAM permissions for admin operations

SHOULD:
- Verify the admin account is not the management account (best practice)
- Check if auto-enable is configured for new member accounts
- Verify the admin account has GuardDuty enabled

MAY:
- Check CloudTrail for admin configuration changes
- Verify no SCP blocking Detective admin operations

## Phase 2 — Remediate

MUST:
- If no admin set, register delegated admin: `aws organizations register-delegated-administrator --account-id <admin-account> --service-principal detective.amazonaws.com`
- If admin needs to change, plan for data loss (graph reset required)
- Ensure admin account has all required permissions

SHOULD:
- Use a dedicated security account as the admin
- Enable auto-enable for Organizations members
- Document the admin account choice and rationale

## Output Format

```yaml
root_cause: "admin_account — <specific_cause>"
evidence:
  - type: delegated_admin
    content: "<delegated administrator details>"
  - type: graph_status
    content: "<behavior graph status>"
severity: HIGH
mitigation:
  immediate: "Configure or fix delegated administrator"
  long_term: "Document admin account architecture and enable auto-enable"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Admin account controls all security investigation data |
| Blast Radius | CRITICAL — Changing admin resets the behavior graph and destroys all data |
| Rollback Complexity | CRITICAL — Admin change is irreversible — all graph data is lost |
| Compliance Scope | CRITICAL — Admin account architecture affects security governance |

## Escalation Conditions

- Escalate to CISO before any admin account changes — data loss is permanent
- Escalate to security architecture team for admin account selection decisions
- Escalate to AWS Support if delegated admin registration fails

## Data Sensitivity

- **Admin account** has access to all member account security data
- **Delegated admin configuration** is a security architecture decision

## PROHIBITED Actions

- Do NOT change the delegated admin account without CISO approval — all data is lost
- Do NOT use the management account as the Detective admin (security best practice)
- Do NOT register delegated admin without understanding the data loss implications

## Phase 3 — Rollback

- Admin changes CANNOT be rolled back — the previous graph and all data are destroyed
- If admin was changed accidentally, create a new graph from the new admin account
- Re-invite all member accounts and re-enable all data sources
- Document the data loss for security audit records

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
