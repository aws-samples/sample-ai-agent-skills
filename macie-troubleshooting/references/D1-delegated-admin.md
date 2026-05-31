---
title: "D1 — Delegated Admin"
description: "Diagnose delegated administrator issues for Macie multi-account"
status: active
severity: HIGH
triggers:
  - "delegated admin error"
  - "cannot manage members"
  - "organization Macie"
  - "admin account"
owner: devops-agent
objective: "Configure and troubleshoot Macie delegated administrator"
context: "Macie supports a delegated administrator that manages Macie for all member accounts in an AWS Organization. Only one delegated admin per organization per region. The management account designates the admin."
---

## Phase 1 — Triage

MUST:
- Check current admin: `aws macie2 get-administrator-account`
- Check from Organizations: `aws organizations list-delegated-administrators --service-principal macie.amazonaws.com`
- Verify Macie is enabled: `aws macie2 get-macie-session`
- List members: `aws macie2 list-members`

SHOULD:
- Verify Organizations trusted access is enabled
- Check if caller is in management or delegated admin account
- Review member account status

MAY:
- Check CloudTrail for admin-related events
- Verify IAM permissions

## Phase 2 — Remediate

MUST:
- Enable Macie if not active: `aws macie2 enable-macie`
- Designate delegated admin (from management account): `aws macie2 enable-organization-admin-account --admin-account-id <account-id>`
- Enable trusted access: `aws organizations enable-aws-service-access --service-principal macie.amazonaws.com`

SHOULD:
- Auto-enable Macie for new member accounts
- Verify admin can see member findings

MAY:
- Configure organization-wide discovery jobs
- Set up cross-account finding aggregation

## Common Issues

- symptoms: "Cannot designate delegated admin"
  diagnosis: "Not in management account or trusted access not enabled."
  resolution: "Run from management account. Enable trusted access."

- symptoms: "Admin cannot see member buckets"
  diagnosis: "Members not associated or Macie not enabled in members."
  resolution: "Associate members and enable Macie in each account."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Macie | YELLOW | Enables service with cost implications |
| Designate delegated admin | RED | Grants organization-wide Macie management |
| Enable trusted access | YELLOW | Grants Macie cross-account permissions |
| Auto-enable for members | YELLOW | Automatically enables Macie in new accounts |

## Escalation Conditions

- Delegated admin designation or changes — escalate to management account administrator and security team
- Macie enablement across organization — escalate to cost management and security team
- Delegated admin account compromise suspected — escalate immediately to security team
- Organization-wide discovery job configuration — escalate to all account owners

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 get-administrator-account` | HIGH | Exposes delegated admin account |
| `aws organizations list-delegated-administrators` | HIGH | Exposes organization admin structure |
| `aws macie2 list-members` | HIGH | Exposes member account IDs |
| `aws macie2 get-macie-session` | LOW | Session status only |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** change delegated admin without management account approval
- **NEVER** disable Macie in member accounts without security team approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled Macie | Disable: `aws macie2 disable-macie` (caution: deletes all findings and jobs) |
| Designated delegated admin | Remove: `aws macie2 disable-organization-admin-account --admin-account-id <id>` |
| Enabled trusted access | Disable: `aws organizations disable-aws-service-access --service-principal macie.amazonaws.com` |

## Output Format

```yaml
root_cause: "delegated_admin — <specific_cause>"
evidence:
  - type: admin_status
    content: "<delegated admin configuration>"
  - type: org_status
    content: "<organization integration status>"
severity: HIGH
mitigation:
  immediate: "Fix delegated admin configuration"
  long_term: "Automate member onboarding"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
