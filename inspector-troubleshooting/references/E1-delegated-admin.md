---
title: "E1 — Delegated Admin"
description: "Diagnose delegated administrator issues for Inspector multi-account"
status: active
severity: HIGH
triggers:
  - "delegated admin error"
  - "cannot manage member accounts"
  - "organization Inspector"
  - "admin account issues"
owner: devops-agent
objective: "Configure and troubleshoot Inspector delegated administrator for Organizations"
context: "Inspector supports a delegated administrator account that manages scanning for all member accounts in an AWS Organization. Only one delegated admin per organization. The management account designates the delegated admin. The admin can enable/disable scanning and view findings across accounts."
---

## Phase 1 — Triage

MUST:
- Check current delegated admin: `aws inspector2 get-delegated-admin-account`
- Check from Organizations: `aws organizations list-delegated-administrators --service-principal inspector2.amazonaws.com`
- Verify the caller is in the management account or delegated admin account
- Check organization integration: `aws inspector2 describe-organization-configuration`

SHOULD:
- Verify Organizations is enabled with all features
- Check if trusted access is enabled for Inspector
- List member accounts: `aws inspector2 list-members --only-associated`

MAY:
- Check CloudTrail for delegation-related events
- Verify IAM permissions for organization management

## Phase 2 — Remediate

MUST:
- Designate delegated admin (from management account): `aws inspector2 enable-delegated-admin-account --delegated-admin-account-id <account-id>`
- Enable auto-enable for new member accounts: `aws inspector2 update-organization-configuration --auto-enable '{"ec2":true,"ecr":true,"lambda":true}'`

SHOULD:
- Enable trusted access: `aws organizations enable-aws-service-access --service-principal inspector2.amazonaws.com`
- Enable scanning for existing member accounts
- Verify member account status after configuration

MAY:
- Set up cross-account finding aggregation
- Configure organization-wide suppression rules

## Common Issues

- symptoms: "Cannot designate delegated admin"
  diagnosis: "Not running from management account or trusted access not enabled."
  resolution: "Run from management account. Enable trusted access for Inspector."

- symptoms: "Delegated admin cannot see member findings"
  diagnosis: "Member accounts not associated or scanning not enabled."
  resolution: "Associate member accounts and enable scanning for each."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Designate delegated admin | RED | Grants organization-wide Inspector management |
| Enable auto-enable for new accounts | YELLOW | Automatically enables scanning in new accounts |
| Enable trusted access | YELLOW | Grants Inspector cross-account permissions |

## Escalation Conditions

- Delegated admin designation or changes — escalate to management account administrator and security team
- Auto-enable configuration changes — escalate to security team
- Delegated admin account compromise suspected — escalate immediately to security team
- Organization-wide scanning changes — escalate to all account owners

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 get-delegated-admin-account` | HIGH | Exposes delegated admin account ID |
| `aws organizations list-delegated-administrators` | HIGH | Exposes organization admin structure |
| `aws inspector2 describe-organization-configuration` | HIGH | Exposes org-wide scanning config |
| `aws inspector2 list-members` | HIGH | Exposes member account IDs and status |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** change delegated admin without management account approval
- **NEVER** disable scanning for member accounts without security team approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Designated delegated admin | Remove: `aws inspector2 disable-delegated-admin-account --delegated-admin-account-id <id>` |
| Enabled auto-enable | Disable: `aws inspector2 update-organization-configuration --auto-enable '{"ec2":false,"ecr":false,"lambda":false}'` |
| Enabled trusted access | Disable: `aws organizations disable-aws-service-access --service-principal inspector2.amazonaws.com` |

## Output Format

```yaml
root_cause: "delegated_admin — <specific_cause>"
evidence:
  - type: admin_status
    content: "<delegated admin configuration>"
  - type: org_config
    content: "<organization configuration>"
severity: HIGH
mitigation:
  immediate: "Fix delegated admin configuration"
  long_term: "Automate member account onboarding"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
