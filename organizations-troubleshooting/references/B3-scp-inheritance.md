---
title: "B3 — SCP Inheritance Issues"
description: "Diagnose SCP inheritance behavior across the OU hierarchy"
status: active
severity: MEDIUM
triggers:
  - "SCP inheritance"
  - "inherited SCP"
  - "parent OU SCP"
  - "SCP not inherited"
  - "effective SCP"
owner: devops-agent
objective: "Understand and fix SCP inheritance behavior across the OU hierarchy"
context: "SCPs are inherited from all parent OUs up to the root. The effective SCP is the intersection of all inherited and directly attached SCPs. A Deny at any level blocks the action for all descendants. Moving an account between OUs changes its effective SCPs immediately."
---

## Phase 1 — Triage

MUST:
- Trace the full OU path from root to account: `aws organizations list-parents --child-id <account-id>` (repeat up to root)
- List SCPs at each level: `aws organizations list-policies-for-target --target-id <target-id> --filter SERVICE_CONTROL_POLICY`
- Get content of each SCP in the hierarchy
- Calculate the effective permissions (intersection of all Allow statements)

SHOULD:
- Check if the FullAWSAccess SCP is attached at every level
- Identify which level in the hierarchy is causing the restriction
- Verify the account's direct SCPs vs inherited SCPs

MAY:
- Compare effective SCPs for accounts in different OUs
- Check for recently moved accounts whose SCPs changed

## Phase 2 — Remediate

MUST:
- Ensure Allow statements at every level include the required actions
- Remove Deny statements at parent levels that block legitimate actions
- Verify the effective permissions after changes

SHOULD:
- Document the OU hierarchy and SCP inheritance chain
- Use a consistent SCP strategy across the hierarchy
- Test SCP changes in a sandbox OU before production

MAY:
- Create a visual diagram of the SCP inheritance tree
- Implement automated SCP compliance checking

## Common Issues

- symptoms: "Action denied but no SCP at the account level blocks it"
  diagnosis: "A parent OU or root SCP is blocking the action through inheritance."
  resolution: "Check SCPs at every level from root to account. Fix the blocking SCP."

- symptoms: "Account moved to new OU — permissions changed unexpectedly"
  diagnosis: "The new OU has different inherited SCPs than the previous OU."
  resolution: "Review SCPs in the new OU hierarchy. Adjust SCPs or move the account back."

- symptoms: "FullAWSAccess removed from a parent OU — all descendants restricted"
  diagnosis: "Without FullAWSAccess (or equivalent), all actions are implicitly denied."
  resolution: "Reattach FullAWSAccess or create an equivalent Allow SCP at the parent level."

## Output Format

```yaml
root_cause: "scp_inheritance_issue — <specific_cause>"
evidence:
  - type: ou_path
    content: "<full OU path from root to account>"
  - type: scp_per_level
    content: "<SCPs at each level>"
  - type: effective_permissions
    content: "<calculated effective permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix the SCP at the appropriate level in the hierarchy"
  long_term: "Document and monitor SCP inheritance across the organization"
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
