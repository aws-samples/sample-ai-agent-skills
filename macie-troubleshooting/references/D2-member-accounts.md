---
title: "D2 — Member Accounts"
description: "Diagnose member account issues in Macie multi-account setup"
status: active
severity: MEDIUM
triggers:
  - "member not enrolled"
  - "account not associated"
  - "member Macie disabled"
  - "cross-account scanning"
owner: devops-agent
objective: "Ensure member accounts are properly enrolled in Macie"
context: "Member accounts must be associated with the delegated admin and have Macie enabled. The admin can enable Macie for members and create discovery jobs across accounts."
---

## Phase 1 — Triage

MUST:
- List members: `aws macie2 list-members --query 'Members[*].{AccountId:AccountId,Status:RelationshipStatus,Email:Email}'`
- Check specific member: `aws macie2 get-member --id <account-id>`
- Verify Macie is enabled in member account

SHOULD:
- Check if auto-enable is configured for new accounts
- Verify member account is part of the organization
- Check invitation status for non-organization members

MAY:
- Review member account IAM permissions
- Check CloudTrail in member account

## Phase 2 — Remediate

MUST:
- Associate member: `aws macie2 create-member --account '{"accountId":"<id>","email":"<email>"}'`
- Enable Macie in member account
- Verify association status

SHOULD:
- Configure auto-enable for new organization accounts
- Batch associate multiple members

MAY:
- Send invitations for non-organization accounts
- Set up cross-account discovery jobs

## Common Issues

- symptoms: "Member shows as INVITED but not ENABLED"
  diagnosis: "Invitation not accepted by member account."
  resolution: "Accept invitation in member account or use organization auto-enable."

- symptoms: "Cannot create discovery jobs for member buckets"
  diagnosis: "Member not properly associated or Macie not enabled."
  resolution: "Verify member association and Macie enablement."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate member account | GREEN | Establishes management relationship |
| Enable Macie in member | YELLOW | Enables service with cost implications |
| Configure auto-enable | YELLOW | Automatically enables Macie in new accounts |

## Escalation Conditions

- Member account not enrolled for compliance — escalate to compliance team
- Batch member enablement — escalate to operations and cost management team
- Member account invitation not accepted — escalate to member account owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 list-members` | HIGH | Exposes member account IDs and status |
| `aws macie2 get-member` | HIGH | Exposes member account details |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** disassociate member accounts without security team approval
- **NEVER** disable Macie in member accounts in production

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Associated member | Disassociate: `aws macie2 disassociate-member --id <account-id>` |
| Enabled Macie in member | Disable Macie in member account |
| Configured auto-enable | Disable auto-enable in organization settings |

## Output Format

```yaml
root_cause: "member_account — <specific_cause>"
evidence:
  - type: member_status
    content: "<member account status>"
severity: MEDIUM
mitigation:
  immediate: "Associate and enable member account"
  long_term: "Configure auto-enable for new accounts"
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
