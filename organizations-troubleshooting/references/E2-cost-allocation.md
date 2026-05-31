---
title: "E2 — Cost Allocation Issues"
description: "Diagnose cost allocation tag and reporting problems"
status: active
severity: LOW
triggers:
  - "cost allocation"
  - "cost allocation tags"
  - "tag not showing in billing"
  - "chargeback"
owner: devops-agent
objective: "Configure cost allocation tags for accurate cost attribution"
context: "Cost allocation tags must be activated at the management account level to appear in billing reports. Tags applied to resources are not automatically cost allocation tags. AWS-generated tags (aws:createdBy) and user-defined tags can both be activated. Tag activation takes up to 24 hours to take effect."
---

## Phase 1 — Triage

MUST:
- Check activated cost allocation tags: `aws ce list-cost-allocation-tags --status Active`
- Check inactive tags: `aws ce list-cost-allocation-tags --status Inactive`
- Verify tags are applied to resources: `aws resourcegroupstaggingapi get-resources --tag-filters Key=<tag-key>`

SHOULD:
- Check if the tag was recently activated (takes up to 24 hours)
- Verify the tag key matches exactly (case-sensitive)
- Check Cost Explorer for tag-based cost breakdown

MAY:
- Review tag policies for enforcement
- Check for untagged resources

## Phase 2 — Remediate

MUST:
- Activate cost allocation tags: done via the Billing console or API
- Ensure tags are consistently applied to resources across accounts
- Wait up to 24 hours for newly activated tags to appear in billing

SHOULD:
- Implement tag policies to enforce consistent tagging
- Use AWS-generated tags (aws:createdBy) for automatic attribution
- Set up tag compliance monitoring

MAY:
- Implement automated tagging via Lambda or Config rules
- Create custom billing reports using cost allocation tags

## Common Issues

- symptoms: "Tags applied to resources but not showing in billing reports"
  diagnosis: "The tag has not been activated as a cost allocation tag."
  resolution: "Activate the tag in the Billing console. Wait 24 hours for it to appear."

- symptoms: "Cost allocation tag shows 'Not tagged' for some resources"
  diagnosis: "The tag is not applied to those specific resources."
  resolution: "Apply the tag to all resources. Use tag policies for enforcement."

## Output Format

```yaml
root_cause: "cost_allocation_issue — <specific_cause>"
evidence:
  - type: active_tags
    content: "<activated cost allocation tags>"
  - type: resource_tags
    content: "<tag application status>"
severity: LOW
mitigation:
  immediate: "Activate cost allocation tags and apply to resources"
  long_term: "Implement tag policies and automated tagging"
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
