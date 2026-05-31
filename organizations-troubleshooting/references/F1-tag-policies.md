---
title: "F1 — Tag Policy Issues"
description: "Diagnose tag policy configuration and enforcement problems"
status: active
severity: MEDIUM
triggers:
  - "tag policy"
  - "tag enforcement"
  - "tag compliance"
  - "tag not enforced"
owner: devops-agent
objective: "Configure tag policies correctly for consistent tagging across the organization"
context: "Tag policies define tagging rules (allowed values, case treatment, enforcement) for resources across the organization. They are inherited through the OU hierarchy. Tag policies can enforce compliance or just report non-compliance. Enforcement prevents non-compliant tag operations."
---

## Phase 1 — Triage

MUST:
- List tag policies: `aws organizations list-policies --filter TAG_POLICY`
- Check effective tag policy: `aws organizations describe-effective-policy --policy-type TAG_POLICY --target-id <account-id>`
- Verify tag policy type is enabled
- Get policy content: `aws organizations describe-policy --policy-id <policy-id>`

SHOULD:
- Check enforcement status (enforced_for vs report-only)
- Check which targets the policy is attached to
- Verify the tag key and allowed values in the policy

MAY:
- Check tag compliance: `aws resourcegroupstaggingapi get-compliance-summary`
- Review non-compliant resources

## Phase 2 — Remediate

MUST:
- Enable tag policy type: `aws organizations enable-policy-type --root-id <root-id> --policy-type TAG_POLICY`
- Create and attach tag policies to appropriate targets
- Verify the effective policy applies correctly

SHOULD:
- Start with report-only mode before enabling enforcement
- Use inheritance operators for flexible policy merging
- Document required tags and allowed values

MAY:
- Implement automated remediation for non-compliant tags
- Use AWS Config rules for additional tag compliance monitoring

## Common Issues

- symptoms: "Tag policy not enforcing — non-compliant tags still allowed"
  diagnosis: "The policy is in report-only mode, not enforcement mode."
  resolution: "Add `enforced_for` to the policy to enable enforcement for specific services."

- symptoms: "Tag policy blocks legitimate tag values"
  diagnosis: "The allowed values list is too restrictive."
  resolution: "Update the tag policy to include the legitimate values."

## Output Format

```yaml
root_cause: "tag_policy_issue — <specific_cause>"
evidence:
  - type: policy
    content: "<tag policy content>"
  - type: effective_policy
    content: "<effective policy for the account>"
  - type: compliance
    content: "<tag compliance status>"
severity: MEDIUM
mitigation:
  immediate: "Fix tag policy configuration"
  long_term: "Implement tag compliance monitoring and remediation"
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
