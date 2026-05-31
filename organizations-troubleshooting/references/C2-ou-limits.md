---
title: "C2 — OU Limits"
description: "Diagnose OU count, nesting, and account limits"
status: active
severity: MEDIUM
triggers:
  - "OU limit"
  - "too many OUs"
  - "nesting limit"
  - "ConstraintViolationException"
owner: devops-agent
objective: "Manage OU limits and optimize the organizational structure"
context: "Organizations has limits on OU count (1,000 per org), nesting depth (5 levels), accounts per OU, and SCPs per target (5). Exceeding these limits prevents further organizational changes."
---

## Phase 1 — Triage

MUST:
- Count total OUs: list all OUs recursively from root
- Check nesting depth of the affected OU path
- Check accounts in the OU: `aws organizations list-children --parent-id <ou-id> --child-type ACCOUNT`
- Check SCPs attached to the target: `aws organizations list-policies-for-target --target-id <target-id> --filter SERVICE_CONTROL_POLICY`

SHOULD:
- Check service quotas: `aws service-quotas get-service-quota --service-code organizations --quota-code L-29A0C5DF`
- Identify unused or empty OUs that can be deleted
- Review the OU hierarchy for optimization opportunities

MAY:
- Request quota increases for account or OU limits
- Compare current usage against limits

## Phase 2 — Remediate

MUST:
- Delete unused OUs to free up capacity
- Restructure the hierarchy to stay within nesting limits
- Consolidate SCPs if the 5-per-target limit is reached

SHOULD:
- Request quota increases proactively before hitting limits
- Use tags instead of deep OU nesting for additional categorization
- Document OU limits and current usage

MAY:
- Implement automated OU cleanup for development/test environments
- Use multiple organizations for very large enterprises (with trade-offs)

## Common Issues

- symptoms: "ConstraintViolationException when creating OU"
  diagnosis: "OU count limit (1,000) or nesting depth limit (5) reached."
  resolution: "Delete unused OUs or restructure the hierarchy."

- symptoms: "Cannot attach more SCPs to an OU"
  diagnosis: "Maximum 5 SCPs per target reached."
  resolution: "Consolidate SCPs or remove unnecessary ones. FullAWSAccess counts as one."

## Output Format

```yaml
root_cause: "ou_limit_issue — <specific_cause>"
evidence:
  - type: ou_count
    content: "<current OU count vs limit>"
  - type: nesting_depth
    content: "<current nesting depth>"
severity: MEDIUM
mitigation:
  immediate: "Delete unused OUs or restructure hierarchy"
  long_term: "Monitor OU usage and request quota increases proactively"
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
