---
title: "A2 — Cost Allocation"
description: "Diagnose cost allocation tag and attribution issues"
status: active
severity: MEDIUM
triggers:
  - "cost allocation not working"
  - "tags not in cost explorer"
  - "cannot group by tag"
  - "cost attribution"
owner: devops-agent
objective: "Resolve cost allocation tag configuration and attribution issues"
context: "Cost allocation tags must be activated in the Billing console separately from resource tags. Activation takes up to 24 hours. Only activated tags appear in Cost Explorer and CUR. AWS-generated tags (aws: prefix) and user-defined tags are activated separately."
---

## Phase 1 — Triage

MUST:
- Check available cost allocation tags: `aws ce get-tags --time-period Start=<start>,End=<end>`
- Verify tags exist on resources: `aws resourcegroupstaggingapi get-resources --tag-filters Key=<tag-key>`
- Check if tags are activated in Billing console (requires console access)
- Verify tag key names match exactly (case-sensitive)

SHOULD:
- Check when tags were activated (only applies to data after activation)
- Verify tag values are consistent across resources
- Review CUR for tag columns: `aws cur describe-report-definitions`

MAY:
- Check for AWS-generated tags (aws:createdBy, etc.)
- Review tag policies in Organizations

## Phase 2 — Remediate

MUST:
- Activate cost allocation tags in Billing console
- Wait 24 hours for activation to take effect
- Ensure resources are tagged with the activated tag keys

SHOULD:
- Implement tag enforcement via AWS Config rules or SCPs
- Standardize tag key names and values across the organization
- Enable AWS-generated cost allocation tags for automatic attribution

MAY:
- Use cost categories as an alternative to tags for cost grouping
- Implement automated tagging via Lambda or CloudFormation

## Common Issues

- symptoms: "Tags not appearing in Cost Explorer"
  diagnosis: "Tags not activated as cost allocation tags in Billing console."
  resolution: "Activate tags in Billing > Cost Allocation Tags. Wait 24 hours."

- symptoms: "Some resources show tag, others don't"
  diagnosis: "Inconsistent tagging across resources."
  resolution: "Audit tagging with Resource Groups Tagging API. Enforce tagging policies."

- symptoms: "Historical costs not tagged"
  diagnosis: "Cost allocation tags only apply to data after activation date."
  resolution: "Tags are not retroactive. Use Cost Explorer filters for historical analysis."

## Output Format

```yaml
root_cause: "cost_allocation — <specific_cause>"
evidence:
  - type: available_tags
    content: "<activated cost allocation tags>"
  - type: resource_tags
    content: "<tag coverage on resources>"
severity: MEDIUM
mitigation:
  immediate: "Activate tags and ensure resources are tagged"
  long_term: "Implement tag enforcement and automated tagging"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Cost allocation tags may reveal organizational structure |
| Blast Radius | LOW — Tag activation is non-destructive; affects reporting only |
| Rollback Complexity | LOW — Tags can be deactivated; but activation is not retroactive |
| Compliance Scope | MEDIUM — Cost allocation supports financial governance |

## Escalation Conditions

- Escalate to finance if cost allocation gaps affect chargeback/showback reporting
- Escalate to operations if tag enforcement is needed across the organization

## Data Sensitivity

- **Tag keys and values** may reveal team names, project codes, or cost centers
- **Cost allocation reports** contain financial data — restrict access appropriately

## PROHIBITED Actions

- Do NOT activate cost allocation tags without coordinating with finance and operations teams
- Do NOT change tag key naming conventions without updating all dependent reports
- Do NOT deactivate cost allocation tags that are used in active financial reports

## Phase 3 — Rollback

- Deactivate incorrectly activated tags in the Billing console
- Note: deactivation only affects future data — historical data retains the tag
- Restore previous tag naming conventions if they were changed
- Notify report consumers of any tag changes

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
