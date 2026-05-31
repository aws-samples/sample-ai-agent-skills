---
title: "A2 — Membership Issues"
description: "Diagnose Clean Rooms membership problems including invitations, acceptance, and abilities"
status: active
severity: HIGH
triggers:
  - "membership"
  - "member invitation"
  - "join collaboration"
  - "membership error"
  - "member abilities"
owner: devops-agent
objective: "Resolve membership issues and ensure all participants can collaborate"
context: "Collaboration members must be invited and accept membership. Each member has specific abilities (CAN_QUERY, CAN_RECEIVE_RESULTS). Common issues include pending invitations, wrong abilities, and membership configuration errors."
---

## Phase 1 — Triage

MUST:
- List memberships: `aws cleanrooms list-memberships`
- Get membership details: `aws cleanrooms get-membership --membership-identifier <id>`
- Check collaboration members: `aws cleanrooms list-members --collaboration-identifier <collab-id>`
- Verify member status (INVITED, ACTIVE, LEFT, REMOVED)

SHOULD:
- Check member abilities match requirements
- Verify the invited account has accepted the membership
- Check if the member has configured tables associated

MAY:
- Check CloudTrail for membership events
- Verify no SCP blocking membership acceptance

## Phase 2 — Remediate

MUST:
- If invitation pending, accept from member account: `aws cleanrooms create-membership --collaboration-identifier <collab-id> --query-log-status ENABLED`
- If wrong abilities, update collaboration member configuration
- If member removed, re-invite

SHOULD:
- Configure default result settings for members who can receive results
- Verify all members have their tables configured after joining

## Output Format

```yaml
root_cause: "membership — <specific_cause>"
evidence:
  - type: membership_status
    content: "<membership details>"
  - type: member_list
    content: "<collaboration members>"
severity: HIGH
mitigation:
  immediate: "Accept invitation or fix member configuration"
  long_term: "Document membership requirements and onboarding process"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Accept membership from member account | YELLOW |
| Update collaboration member configuration | YELLOW |
| Re-invite removed member | GREEN |
| Configure default result settings | GREEN |
| Verify all members have tables configured | GREEN |

## Escalation Conditions

- Fix requires modifying production collaboration memberships
- Fix involves changing member abilities that affect data access

## Data Sensitivity

- **HIGH**: membership configurations expose account relationships and data sharing permissions
- Member abilities determine what data operations each participant can perform

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If membership was accepted incorrectly, leave the collaboration: `aws cleanrooms delete-membership --membership-identifier <id>`
2. If member abilities were changed, restore previous configuration
3. If result settings were modified, revert to previous output configuration
4. Verify rollback by confirming `aws cleanrooms list-members` shows expected membership state

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
