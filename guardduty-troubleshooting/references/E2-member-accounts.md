---
title: "E2 — Member Account Issues"
description: "Diagnose GuardDuty member account management and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "member account"
  - "member not associated"
  - "member account guardduty"
  - "account invitation"
  - "member detector"
owner: devops-agent
objective: "Identify and resolve member account association and configuration issues"
context: "Member accounts are managed by the delegated admin or administrator account. Issues include failed associations, member accounts not accepting invitations, feature mismatches between admin and member, and member accounts leaving the organization."
---

## Phase 1 — Triage

MUST:
- List member accounts: `aws guardduty list-members --detector-id <detector-id>`
- Check member status (INVITED, ENABLED, DISABLED, RESIGNED)
- Get member detector details: `aws guardduty get-member-detectors --detector-id <detector-id> --account-ids <account-id>`
- Verify the member account is in the organization: `aws organizations list-accounts`

SHOULD:
- Check for pending invitations: `aws guardduty list-members --detector-id <detector-id> --only-associated false`
- Verify organization configuration auto-enable settings: `aws guardduty describe-organization-configuration --detector-id <detector-id>`
- Check CloudTrail for member management events

MAY:
- Check if the member account has its own detector configuration
- Verify no SCPs block GuardDuty in the member account

## Phase 2 — Remediate

MUST:
- Associate missing member accounts: `aws guardduty create-members --detector-id <detector-id> --account-details '[{"AccountId":"<id>","Email":"<email>"}]'`
- Enable auto-enable for new accounts in organization configuration
- Verify member detector features match admin expectations

SHOULD:
- Re-invite members that have RESIGNED status
- Update member detector features to match organization policy

MAY:
- Set up monitoring for member account status changes
- Automate member account onboarding

## Common Issues

- symptoms: "Member account shows INVITED but not ENABLED"
  diagnosis: "The member account has not accepted the invitation (for invitation-based association)."
  resolution: "For organization-based management, use auto-enable. For invitation-based, the member must accept."

- symptoms: "Member account features differ from admin configuration"
  diagnosis: "The member was added before certain features were enabled in the organization configuration."
  resolution: "Update member detector features or update organization configuration and re-apply."

- symptoms: "Member account left the organization and GuardDuty is disabled"
  diagnosis: "When an account leaves the organization, it is disassociated from the admin and its detector may be disabled."
  resolution: "If the account rejoins, re-associate it. If standalone, the account must manage its own GuardDuty."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate member accounts | YELLOW | Organization change; reversible |
| Update member detector features | YELLOW | Configuration change; reversible |
| Re-invite members | YELLOW | Invitation; reversible |

## Escalation Conditions

- Member account association changes
- Detector configuration changes across accounts

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-members` | MEDIUM | Member account IDs and status |
| `get-member-detectors` | MEDIUM | Member feature configuration |
| `describe-organization-configuration` | MEDIUM | Org-wide settings |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Member association | Disassociate via `disassociate-members` |
| Feature update | Revert via `update-member-detectors` |

## Output Format

```yaml
root_cause: "member_account — <specific_cause>"
evidence:
  - type: members
    content: "<list-members output>"
  - type: member_detectors
    content: "<member detector feature status>"
  - type: org_config
    content: "<organization auto-enable settings>"
severity: MEDIUM
mitigation:
  immediate: "Associate or re-invite the member account"
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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
