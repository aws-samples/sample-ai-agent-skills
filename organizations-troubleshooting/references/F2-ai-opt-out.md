---
title: "F2 — AI Opt-Out Policy Issues"
description: "Diagnose AI services opt-out policy configuration and enforcement"
status: active
severity: LOW
triggers:
  - "AI opt-out"
  - "AI services policy"
  - "opt out AI"
  - "machine learning opt out"
owner: devops-agent
objective: "Configure AI opt-out policies correctly across the organization"
context: "AI opt-out policies control whether AWS AI services can use customer content for service improvement. Policies are attached to OUs or accounts and inherited through the hierarchy. They apply to specific AI services (CodeWhisperer, Lex, Comprehend, etc.)."
---

## Phase 1 — Triage

MUST:
- List AI opt-out policies: `aws organizations list-policies --filter AISERVICES_OPT_OUT_POLICY`
- Check effective policy for the account: `aws organizations describe-effective-policy --policy-type AISERVICES_OPT_OUT_POLICY --target-id <account-id>`
- Verify AI opt-out policies are enabled for the organization
- Get policy content: `aws organizations describe-policy --policy-id <policy-id>`

SHOULD:
- Check which targets the policy is attached to: `aws organizations list-targets-for-policy --policy-id <policy-id>`
- Verify the policy covers the desired AI services

MAY:
- Compare effective policies across different accounts/OUs

## Phase 2 — Remediate

MUST:
- Enable AI opt-out policy type if not enabled: `aws organizations enable-policy-type --root-id <root-id> --policy-type AISERVICES_OPT_OUT_POLICY`
- Create and attach the opt-out policy to the appropriate targets
- Verify the effective policy applies to the correct accounts

SHOULD:
- Apply opt-out policies at the root level for organization-wide coverage
- Document which AI services are opted out
- Review policies when new AI services are launched

MAY:
- Use different opt-out policies for different OUs (e.g., opt out in production, allow in sandbox)

## Common Issues

- symptoms: "AI opt-out policy not taking effect"
  diagnosis: "The policy type is not enabled for the organization."
  resolution: "Enable AISERVICES_OPT_OUT_POLICY type at the root level."

- symptoms: "Policy attached but specific AI service not opted out"
  diagnosis: "The policy content does not include the specific AI service."
  resolution: "Update the policy to include the desired AI service opt-out."

## Output Format

```yaml
root_cause: "ai_opt_out_issue — <specific_cause>"
evidence:
  - type: policy
    content: "<AI opt-out policy content>"
  - type: effective_policy
    content: "<effective policy for the account>"
severity: LOW
mitigation:
  immediate: "Enable and configure AI opt-out policies"
  long_term: "Review policies when new AI services are launched"
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
