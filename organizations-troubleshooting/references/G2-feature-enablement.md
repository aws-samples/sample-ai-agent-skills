---
title: "G2 — Feature Enablement Issues"
description: "Diagnose organization feature and policy type enablement problems"
status: active
severity: MEDIUM
triggers:
  - "enable feature"
  - "policy type not enabled"
  - "all features"
  - "enable SCP"
  - "enable tag policy"
owner: devops-agent
objective: "Enable required organization features and policy types"
context: "Organizations can operate in 'consolidated billing only' or 'all features' mode. Policy types (SCP, tag policy, backup policy, AI opt-out) must be individually enabled at the root level. Enabling 'all features' is a one-way upgrade. Each policy type requires explicit enablement."
---

## Phase 1 — Triage

MUST:
- Check organization feature set: `aws organizations describe-organization --query 'Organization.FeatureSet'`
- List enabled policy types: `aws organizations list-roots --query 'Roots[0].PolicyTypes'`
- Verify the caller is in the management account
- Check if the required policy type is enabled

SHOULD:
- Check for pending handshakes related to feature enablement: `aws organizations list-handshakes-for-organization`
- Review CloudTrail for enablement events

MAY:
- Check if all member accounts have accepted the all-features invitation

## Phase 2 — Remediate

MUST:
- Enable all features if needed: `aws organizations enable-all-features` (requires member account acceptance)
- Enable specific policy types: `aws organizations enable-policy-type --root-id <root-id> --policy-type <type>`
- Verify the policy type appears in the enabled list

SHOULD:
- Document which features and policy types are enabled
- Plan feature enablement carefully (all features is irreversible)

MAY:
- Use AWS Control Tower for automated feature management
- Set up monitoring for feature and policy type changes

## Common Issues

- symptoms: "Cannot create SCPs — policy type not enabled"
  diagnosis: "SERVICE_CONTROL_POLICY type is not enabled at the root."
  resolution: "Enable SCP type: `aws organizations enable-policy-type --root-id <root-id> --policy-type SERVICE_CONTROL_POLICY`"

- symptoms: "Cannot enable all features — pending handshakes"
  diagnosis: "Member accounts must accept the all-features invitation."
  resolution: "Check pending handshakes and have member accounts accept."

- symptoms: "Cannot downgrade from all features to consolidated billing"
  diagnosis: "All features mode is a one-way upgrade."
  resolution: "This cannot be reversed. Detach SCPs if you don't want enforcement."

## Output Format

```yaml
root_cause: "feature_enablement_issue — <specific_cause>"
evidence:
  - type: feature_set
    content: "<organization feature set>"
  - type: policy_types
    content: "<enabled policy types>"
severity: MEDIUM
mitigation:
  immediate: "Enable the required feature or policy type"
  long_term: "Document enabled features and plan changes carefully"
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
