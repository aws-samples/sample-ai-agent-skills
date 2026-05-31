---
title: "B2 — SCP Conflicts"
description: "Diagnose conflicts between multiple SCPs and resolve unexpected behavior"
status: active
severity: HIGH
triggers:
  - "SCP conflict"
  - "multiple SCPs"
  - "SCP contradiction"
  - "overlapping SCPs"
owner: devops-agent
objective: "Identify and resolve conflicts between multiple SCPs"
context: "Multiple SCPs can be attached at different levels (root, OU, account). When SCPs conflict, Deny always wins. If multiple Allow SCPs exist, the effective allow is the intersection. Understanding SCP conflict resolution is critical for correct permission boundaries."
---

## Phase 1 — Triage

MUST:
- List all SCPs at every level from root to account
- Get content of each SCP: `aws organizations describe-policy --policy-id <policy-id>`
- Identify conflicting statements (Allow vs Deny for the same action)
- Check for implicit denies (action not in any Allow statement)

SHOULD:
- Map the complete SCP hierarchy for the affected account
- Check for condition keys that may cause conditional conflicts
- Verify the FullAWSAccess SCP is present at each level

MAY:
- Create a matrix of actions vs SCPs to visualize conflicts
- Test specific actions from the affected account

## Phase 2 — Remediate

MUST:
- Remove or modify conflicting Deny statements
- Ensure Allow statements at each level include required actions
- Verify the fix from a member account

SHOULD:
- Consolidate SCPs where possible to reduce complexity
- Use explicit Deny only for actions that must be blocked
- Document the SCP strategy and conflict resolution

MAY:
- Use AWS Access Analyzer to validate SCP effects
- Implement SCP testing in a sandbox OU

## Common Issues

- symptoms: "Action allowed in one OU but denied in another"
  diagnosis: "Different SCPs attached to different OUs."
  resolution: "Review SCPs at each OU level. Ensure consistent policy across the hierarchy."

- symptoms: "SCP Allow at OU level but Deny at root level"
  diagnosis: "Deny at root overrides Allow at OU. Deny always wins."
  resolution: "Remove the Deny from the root SCP or add an exception condition."

## Output Format

```yaml
root_cause: "scp_conflict — <specific_cause>"
evidence:
  - type: scp_hierarchy
    content: "<all SCPs at each level>"
  - type: conflicting_statements
    content: "<specific conflicting Allow/Deny statements>"
severity: HIGH
mitigation:
  immediate: "Resolve the SCP conflict"
  long_term: "Document SCP strategy and implement change management"
```


## Safety Ratings
- GREEN: describe-organization, list-accounts, list-policies
- YELLOW: attach-policy, create-organizational-unit, move-account
- RED: close-account, delete-policy, remove-account-from-organization

## Escalation Conditions
- "Remediation involves modifying SCPs in production"
- "Fix requires moving accounts between OUs"
- "Change affects the management account"
- "SCP modification could lock out member accounts"

## Rollback
- Pre-change: "Document current SCP attachments and OU structure"
- Verification: "Test affected account access after SCP change"
- Revert: "Detach modified SCP and reattach original"

## Data Sensitivity
- HIGH: "list-accounts reveals all account IDs, names, and emails"
- HIGH: "SCP content reveals security boundaries"
- MEDIUM: "OU structure reveals organizational hierarchy"

## Prohibited Actions
- NEVER suggest removing all SCPs from an OU
- NEVER suggest closing an account without confirming data backup
- NEVER suggest modifying the management account's policies
- NEVER suggest removing an account from the organization without root email access confirmation

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
  - command: "list-accounts"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-organization"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-policies"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest removing SCPs without understanding blast radius"
  - "NEVER suggest disabling AI opt-out policies without review"
  - "NEVER modify organization structure without approval"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
