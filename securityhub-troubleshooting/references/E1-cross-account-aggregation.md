---
title: "E1 — Cross-Account Aggregation"
description: "Diagnose Security Hub cross-account finding aggregation issues"
status: active
severity: HIGH
triggers:
  - "cross-account aggregation"
  - "member account findings"
  - "multi-account security hub"
  - "centralized findings"
  - "admin member"
owner: devops-agent
objective: "Identify and resolve cross-account finding aggregation issues in Security Hub"
context: "Security Hub supports multi-account management through admin-member relationships. The admin account can view and manage findings from all member accounts. Issues arise from member association failures, invitation acceptance, or organization configuration."
---

## Phase 1 — Triage

MUST:
- List member accounts: `aws securityhub list-members`
- Check member status (INVITED, ENABLED, DISABLED, RESIGNED)
- Verify the admin account: `aws securityhub get-administrator-account`
- Check organization configuration: `aws securityhub describe-organization-configuration`
- Verify findings from member accounts: `aws securityhub get-findings --filters '{"AwsAccountId":[{"Value":"<member-account-id>","Comparison":"EQUALS"}]}' --max-results 10`

SHOULD:
- Check Organizations access: `aws organizations describe-organization`
- Verify auto-enable settings for new member accounts
- Check for pending invitations

MAY:
- Check CloudTrail for member management events
- Verify Security Hub is enabled in member accounts

## Phase 2 — Remediate

MUST:
- Associate missing member accounts
- Enable auto-enable for new accounts in organization configuration
- Verify findings from member accounts appear in the admin account

SHOULD:
- Use organization-based management for simpler member management
- Enable Security Hub in all member accounts

MAY:
- Set up monitoring for member account status
- Create insights for cross-account finding analysis

## Common Issues

- symptoms: "Member account findings not visible in admin account"
  diagnosis: "The member account is not associated, has not accepted the invitation, or Security Hub is not enabled in the member."
  resolution: "Associate the member account. For organization-based, enable auto-enable. Verify Security Hub is enabled in the member."

- symptoms: "Cannot associate member account — already associated with another admin"
  diagnosis: "A member account can only be associated with one admin account."
  resolution: "Disassociate the member from the current admin before associating with the new admin."

- symptoms: "New accounts joining the organization not automatically getting Security Hub"
  diagnosis: "Auto-enable is not configured in the organization configuration."
  resolution: "Enable auto-enable in the organization configuration for new member accounts."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate member accounts | YELLOW | Organization change; reversible |
| Enable auto-enable | YELLOW | Organization change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Member account association changes
- Organization configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-members` | MEDIUM | Member account IDs and status |
| `describe-organization-configuration` | MEDIUM | Org-wide settings |
| `get-findings` (cross-account) | HIGH | Security findings from member accounts |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Member association | Disassociate member accounts |
| Auto-enable | Disable auto-enable in organization configuration |

## Output Format

```yaml
root_cause: "cross_account_aggregation — <specific_cause>"
evidence:
  - type: members
    content: "<member account status>"
  - type: org_config
    content: "<organization configuration>"
  - type: findings
    content: "<cross-account finding visibility>"
severity: HIGH
mitigation:
  immediate: "Associate member accounts and verify finding visibility"
  long_term: "Configure auto-enable and monitor member account status"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
