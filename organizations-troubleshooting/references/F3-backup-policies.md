---
title: "F3 — Backup Policy Issues"
description: "Diagnose organization-level backup policy configuration and enforcement"
status: active
severity: MEDIUM
triggers:
  - "backup policy"
  - "organization backup"
  - "centralized backup"
  - "backup policy inheritance"
owner: devops-agent
objective: "Configure organization backup policies correctly for centralized backup management"
context: "Organization backup policies define backup plans that are applied across member accounts. They use AWS Backup and are inherited through the OU hierarchy. Effective backup policies are merged from all levels. The management account must enable backup policy type."
---

## Phase 1 — Triage

MUST:
- List backup policies: `aws organizations list-policies --filter BACKUP_POLICY`
- Check effective backup policy: `aws organizations describe-effective-policy --policy-type BACKUP_POLICY --target-id <account-id>`
- Verify backup policy type is enabled
- Get policy content: `aws organizations describe-policy --policy-id <policy-id>`

SHOULD:
- Check which targets the policy is attached to
- Verify AWS Backup is configured in member accounts
- Check for policy merge conflicts between levels

MAY:
- Compare effective policies across accounts
- Check AWS Backup job status in member accounts

## Phase 2 — Remediate

MUST:
- Enable backup policy type: `aws organizations enable-policy-type --root-id <root-id> --policy-type BACKUP_POLICY`
- Create and attach backup policies to appropriate targets
- Verify effective policies in member accounts

SHOULD:
- Use inheritance operators (@@assign, @@append, @@remove) for flexible policy merging
- Test backup policies in a sandbox OU first
- Monitor backup job success rates across the organization

MAY:
- Use delegated administrator for backup management
- Implement compliance reporting for backup coverage

## Common Issues

- symptoms: "Backup policy not creating backup plans in member accounts"
  diagnosis: "Backup policy type not enabled or policy not attached to the correct target."
  resolution: "Enable BACKUP_POLICY type and attach the policy to the root or appropriate OU."

- symptoms: "Backup policy merge produces unexpected results"
  diagnosis: "Inheritance operators (@@assign, @@append) are not used correctly."
  resolution: "Review the effective policy and adjust inheritance operators."

## Output Format

```yaml
root_cause: "backup_policy_issue — <specific_cause>"
evidence:
  - type: policy
    content: "<backup policy content>"
  - type: effective_policy
    content: "<effective policy for the account>"
severity: MEDIUM
mitigation:
  immediate: "Enable and configure backup policies"
  long_term: "Monitor backup compliance across the organization"
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
