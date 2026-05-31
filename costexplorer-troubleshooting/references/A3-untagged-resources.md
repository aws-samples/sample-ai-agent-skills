---
title: "A3 — Untagged Resources"
description: "Identify and remediate untagged resources causing cost allocation gaps"
status: active
severity: MEDIUM
triggers:
  - "untagged resources"
  - "missing tags"
  - "cost not attributed"
  - "tag compliance"
owner: devops-agent
objective: "Find and tag untagged resources for proper cost allocation"
context: "Untagged resources create gaps in cost allocation, making it impossible to attribute costs to teams, projects, or environments. AWS Config rules and Resource Groups Tagging API help identify untagged resources."
---

## Phase 1 — Triage

MUST:
- Find untagged resources: `aws resourcegroupstaggingapi get-resources --tags-per-page 100 --resource-type-filters <type>` (filter for resources without specific tags)
- Check cost of untagged resources in Cost Explorer by filtering for empty tag values
- Identify resource types with lowest tag coverage
- Check AWS Config for tag compliance rules: `aws configservice describe-compliance-by-config-rule --config-rule-names required-tags`

SHOULD:
- List resources without required tags: `aws configservice get-compliance-details-by-config-rule --config-rule-name required-tags --compliance-types NON_COMPLIANT`
- Check which accounts have the most untagged resources
- Review tag policies in Organizations: `aws organizations describe-effective-policy --policy-type TAG_POLICY --target-id <account-id>`

MAY:
- Generate a report of all untagged resources by service
- Check if automated tagging is configured

## Phase 2 — Remediate

MUST:
- Tag high-cost untagged resources immediately
- Implement required-tags AWS Config rule for enforcement
- Document required tag keys and values

SHOULD:
- Set up automated tagging for new resources via CloudFormation/Terraform defaults
- Create tag policies in Organizations for standardization
- Implement Lambda-based auto-tagging for resources created without tags

MAY:
- Set up SCPs to prevent resource creation without required tags
- Create dashboards for tag compliance monitoring

## Common Issues

- symptoms: "Large portion of costs unattributed"
  diagnosis: "Many resources missing required cost allocation tags."
  resolution: "Audit with Resource Groups Tagging API. Bulk-tag resources."

- symptoms: "New resources consistently untagged"
  diagnosis: "No enforcement mechanism for tagging at creation time."
  resolution: "Implement AWS Config required-tags rule and SCP enforcement."

- symptoms: "Tags inconsistent across teams"
  diagnosis: "No standardized tagging policy."
  resolution: "Create and enforce tag policies via Organizations."

## Output Format

```yaml
root_cause: "untagged_resources — <specific_cause>"
evidence:
  - type: tag_compliance
    content: "<tag coverage statistics>"
  - type: untagged_costs
    content: "<cost of untagged resources>"
severity: MEDIUM
mitigation:
  immediate: "Tag high-cost resources and enable Config rules"
  long_term: "Implement tag policies, enforcement, and automated tagging"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Resource inventory reveals infrastructure details |
| Blast Radius | LOW — Tagging is non-destructive; does not affect resource behavior |
| Rollback Complexity | LOW — Tags can be removed if incorrectly applied |
| Compliance Scope | MEDIUM — Tag compliance supports governance and cost attribution |

## Escalation Conditions

- Escalate to operations if large portions of infrastructure are untagged
- Escalate to management if untagged resources prevent accurate cost attribution

## Data Sensitivity

- **Resource inventories** reveal infrastructure details — restrict access
- **Tag compliance reports** may expose team-level compliance gaps

## PROHIBITED Actions

- Do NOT bulk-tag resources without validating tag values against the tagging standard
- Do NOT enforce tagging SCPs without a grace period for existing resources
- Do NOT delete resources solely because they are untagged

## Phase 3 — Rollback

- Remove incorrectly applied tags: `aws resourcegroupstaggingapi untag-resources --resource-arn-list <arns> --tag-keys <keys>`
- Revert SCP changes if tag enforcement blocks legitimate resource creation
- Restore previous Config rule settings if they were modified

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
