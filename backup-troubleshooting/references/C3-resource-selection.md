---
title: "C3 — Resource Selection Issues"
description: "Diagnose why backup plans are not selecting the expected resources"
status: active
severity: MEDIUM
triggers:
  - "resource not backed up"
  - "tag selection"
  - "resource selection"
  - "missing resources"
  - "backup not covering"
owner: devops-agent
objective: "Identify why backup plan resource selections are not matching expected resources"
context: "Backup selections can target resources by tag conditions or specific ARNs. Tag-based selection is dynamic (auto-includes new matching resources). ARN-based is static. Tag conditions are case-sensitive and support StringEquals and StringLike operators. Resources can be excluded by conditions."
---

## Phase 1 — Triage

MUST:
- Get the backup selection: `aws backup get-backup-selection --backup-plan-id <plan-id> --selection-id <selection-id>`
- Check the selection's Resources (ARNs), ListOfTags (tag conditions), and NotResources (exclusions)
- Verify the target resources have the expected tags: `aws resourcegroupstaggingapi get-resources --tag-filters Key=<key>,Values=<value>`
- Check if the resource type is supported by AWS Backup
- List all selections for the plan: `aws backup list-backup-selections --backup-plan-id <plan-id>`

SHOULD:
- Verify tag values are exact matches (case-sensitive for StringEquals)
- Check if resources are excluded by NotResources or Conditions
- Verify the IAM role in the selection has permissions for the resource types
- Check if resources are in the same region as the backup plan

MAY:
- Check if resources were recently created (tag-based selection picks them up on next schedule)
- Verify no other backup plan is already backing up the same resources
- Check for tag propagation delays on newly tagged resources

## Phase 2 — Remediate

MUST:
- Fix tag conditions to match actual resource tags
- Update ARN lists if using ARN-based selection
- Ensure the selection's IAM role has permissions for all targeted resource types

SHOULD:
- Use tag-based selection for dynamic environments
- Use consistent tagging standards across resources
- Test selection by running an on-demand backup

MAY:
- Use AWS Config rules to enforce backup tagging compliance
- Implement tag policies via AWS Organizations for consistency

## Common Issues

- symptoms: "New resources are not being backed up"
  diagnosis: "Resources are selected by ARN (static) instead of tags (dynamic)."
  resolution: "Switch to tag-based selection for dynamic resource inclusion."

- symptoms: "Tag-based selection misses resources with correct tags"
  diagnosis: "Tag values are case-sensitive. 'Production' != 'production' for StringEquals."
  resolution: "Use StringLike with wildcards for case-insensitive matching, or standardize tag values."

- symptoms: "Resource backed up by multiple plans"
  diagnosis: "Resource matches selections in multiple backup plans (tag + ARN overlap)."
  resolution: "Consolidate selections or use exclusion conditions to prevent overlap."

- symptoms: "Selection includes unsupported resource types"
  diagnosis: "Tag-based selection with wildcard resource type includes unsupported services."
  resolution: "Specify supported resource types in the selection or use resource type conditions."

## Output Format

```yaml
root_cause: "resource_selection — <specific_cause>"
evidence:
  - type: selection
    content: "<backup selection configuration>"
  - type: resources
    content: "<actual resources and their tags>"
  - type: matching
    content: "<which resources match vs expected>"
severity: MEDIUM
mitigation:
  immediate: "Fix selection criteria to match intended resources"
  long_term: "Implement tagging standards and backup coverage monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix tag conditions to match actual tags | YELLOW | Changes which resources are backed up |
| Update ARN lists | YELLOW | Changes which resources are backed up |
| Ensure selection IAM role has permissions | YELLOW | IAM policy change |
| Use tag-based selection for dynamic environments | YELLOW | Changes selection strategy |
| Test selection with on-demand backup | GREEN | Testing — non-destructive |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Selection changes may leave resources unprotected
- Tag condition changes affect dynamic resource inclusion

## Data Sensitivity

- Resource selection reveals which resources are under backup protection
- Tag conditions reveal resource classification and organization
- ARN lists reveal specific infrastructure components

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER narrow resource selection without verifying all resources remain protected
- NEVER remove tag-based selection without understanding dynamic resource implications

## Phase 3 — Rollback

- If tag conditions were changed: restore original tag filter conditions
- If ARN lists were modified: restore original ARN list
- If IAM role was changed: revert to previous IAM policy
- Document all selection changes for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
