---
title: "C2 — Member Invitation Issues"
description: "Diagnose Detective member account invitation and enrollment problems"
status: active
severity: MEDIUM
triggers:
  - "member invitation"
  - "invite member"
  - "member not joined"
  - "invitation pending"
  - "add member account"
owner: devops-agent
objective: "Resolve member invitation issues and ensure all accounts are enrolled"
context: "Detective member accounts contribute data to the behavior graph. Members can be added via invitation (manual accept) or Organizations auto-enable (automatic). Common issues include pending invitations, invitation limits, and auto-enable configuration."
---

## Phase 1 — Triage

MUST:
- List members and their status: `aws detective list-members --graph-arn <graph-arn>`
- Check for pending invitations: `aws detective list-invitations` (from member account)
- Verify member count against 1200 limit
- Check member status (INVITED, VERIFICATION_IN_PROGRESS, ENABLED)

SHOULD:
- Verify Organizations auto-enable is configured if applicable
- Check if the member account has GuardDuty enabled
- Verify email addresses for invitation delivery

MAY:
- Check CloudTrail for invitation events
- Verify no SCP blocking Detective member operations

## Phase 2 — Remediate

MUST:
- If invitation pending, accept from member account: `aws detective accept-invitation --graph-arn <graph-arn>`
- If using Organizations, enable auto-enable: `aws detective update-organization-configuration --graph-arn <graph-arn> --auto-enable`
- If member limit reached, prioritize accounts by risk

SHOULD:
- Use Organizations auto-enable for automatic enrollment
- Verify member data sources are enabled after joining

## Output Format

```yaml
root_cause: "member_invitation — <specific_cause>"
evidence:
  - type: member_list
    content: "<member status list>"
  - type: invitation_status
    content: "<pending invitations>"
severity: MEDIUM
mitigation:
  immediate: "Accept pending invitations or enable auto-enable"
  long_term: "Use Organizations auto-enable for all accounts"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Member enrollment shares account security data with the admin |
| Blast Radius | LOW — Invitation changes affect individual accounts |
| Rollback Complexity | LOW — Members can be removed from the graph |
| Compliance Scope | HIGH — All accounts should be enrolled for complete security coverage |

## Escalation Conditions

- Escalate to security team if critical accounts are not enrolled
- Escalate to account owners if invitations are pending acceptance
- Escalate to management if member limit (1200) is approaching

## Data Sensitivity

- **Member account IDs** reveal organizational structure
- **Invitation status** is operational data

## PROHIBITED Actions

- Do NOT add accounts to the behavior graph without account owner awareness
- Do NOT remove member accounts during active security investigations
- Do NOT ignore pending invitations for security-critical accounts

## Phase 3 — Rollback

- Remove incorrectly added members: `aws detective delete-members --graph-arn <arn> --account-ids <ids>`
- Note: removing a member deletes their data from the graph
- Re-invite members if they were accidentally removed

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
