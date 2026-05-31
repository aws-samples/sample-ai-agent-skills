---
title: "B1 — SCP Evaluation Issues"
description: "Diagnose SCP evaluation logic and unexpected permission behavior"
status: active
severity: HIGH
triggers:
  - "SCP not working"
  - "SCP evaluation"
  - "SCP deny not blocking"
  - "SCP allow not permitting"
  - "service control policy"
owner: devops-agent
objective: "Understand SCP evaluation logic and fix unexpected permission behavior"
context: "SCPs are deny filters that define the maximum permissions boundary. They do NOT grant permissions. The effective permissions for a member account are the intersection of all inherited SCPs and IAM grants. The management account is exempt from SCPs. SCP evaluation follows: if any SCP denies, the action is blocked; if no SCP allows, the action is implicitly denied."
---

## Phase 1 — Triage

MUST:
- Verify the account is a member account (not management): `aws organizations describe-account --account-id <account-id>`
- List SCPs attached to the account: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`
- List SCPs attached to the account's OU: `aws organizations list-parents --child-id <account-id>` then `aws organizations list-policies-for-target --target-id <ou-id> --filter SERVICE_CONTROL_POLICY`
- Get the full SCP content: `aws organizations describe-policy --policy-id <policy-id>`
- Check SCPs at every level up to root

SHOULD:
- Simulate the IAM policy: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <action>`
- Check for explicit Deny statements in any SCP in the hierarchy
- Verify the FullAWSAccess SCP is attached (or equivalent Allow)

MAY:
- Check for condition keys in SCPs that may affect evaluation
- Review CloudTrail for the denied event details

## Phase 2 — Remediate

MUST:
- Ensure at least one SCP at each level allows the required action
- Remove or modify explicit Deny statements that block legitimate actions
- Verify the fix from a member account (not the management account)

SHOULD:
- Document the SCP hierarchy and effective permissions
- Test SCP changes in a sandbox OU before applying to production
- Use the IAM policy simulator to validate SCP effects

MAY:
- Create a visual map of the SCP inheritance hierarchy
- Implement SCP change management with approval workflows

## Common Issues

- symptoms: "SCP Deny is not blocking the action"
  diagnosis: "Testing from the management account, which is exempt from SCPs."
  resolution: "Test from a member account. The management account is always exempt."

- symptoms: "SCP Allow is attached but action is still denied"
  diagnosis: "SCP Allow does not grant permissions — it only permits IAM policies to grant them."
  resolution: "Ensure the IAM policy also grants the action. SCPs are filters, not grants."

- symptoms: "Action blocked but no explicit Deny in any SCP"
  diagnosis: "The action is not included in any SCP Allow statement (implicit deny)."
  resolution: "Add the action to an SCP Allow statement at every level in the hierarchy."

- symptoms: "SCP works for some accounts but not others in the same OU"
  diagnosis: "Account-level SCPs differ, or the account has additional SCPs attached directly."
  resolution: "Check SCPs at the account level, not just the OU level."

## Output Format

```yaml
root_cause: "scp_evaluation_issue — <specific_cause>"
evidence:
  - type: scp_hierarchy
    content: "<SCPs at each level from root to account>"
  - type: scp_content
    content: "<relevant SCP statements>"
  - type: account_type
    content: "<member or management account>"
severity: HIGH
mitigation:
  immediate: "Fix the SCP to allow or deny the correct actions"
  long_term: "Document SCP hierarchy and test changes in sandbox"
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
