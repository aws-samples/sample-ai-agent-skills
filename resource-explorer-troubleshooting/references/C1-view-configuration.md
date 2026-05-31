---
title: "C1 — View Configuration"
description: "Diagnose view configuration issues in Resource Explorer"
status: active
severity: MEDIUM
triggers: ["view error", "view not working", "cannot create view", "view shows wrong results"]
owner: devops-agent
objective: "Resolve view configuration issues affecting search results"
context: "Views filter search results. Default view shows all resources. Custom views can include filters and property selections."
---

## Phase 1 — Triage

MUST:
- List all views: `aws resource-explorer-2 list-views`
- Get view details: `aws resource-explorer-2 get-view --view-arn <view-arn>`
- Test search with default view vs custom view to isolate the issue

SHOULD:
- Check view filters for incorrect expressions
- Verify the view is associated with the correct index

MAY:
- Compare results between different views
- Review view creation history in CloudTrail

## Phase 2 — Remediate

MUST:
- Fix filter expressions in the view configuration
- Ensure the view includes the required resource properties
- Verify the view is associated with an ACTIVE index

SHOULD:
- Test view changes with a small query before broad use
- Document view purposes and filter logic

MAY:
- Create separate views for different teams or use cases
- Set up the default view with commonly needed properties

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 list-views: GREEN — read-only view enumeration"
  - "resource-explorer-2 get-view: GREEN — read-only view details"
  - "resource-explorer-2 update-view: YELLOW — modifies view configuration"
  - "resource-explorer-2 create-view: YELLOW — creates new view"
```

## Escalation Conditions

- Default view returns no results despite ACTIVE index
- View creation fails with permission errors
- View filters cannot express the required search criteria
- Multiple teams depend on a view that needs modification

## Data Sensitivity

- `aws resource-explorer-2 list-views`: LOW — returns view ARNs and names
- `aws resource-explorer-2 get-view`: LOW — returns view configuration
- View filter expressions: LOW — reveals search criteria

## Prohibited Actions

- NEVER delete the default view without creating a replacement
- NEVER modify shared views without notifying dependent teams
- NEVER assume views provide security boundaries — they only filter results
- NEVER use views as a substitute for IAM-based access control

## Phase 3 — Rollback

1. If a view was modified incorrectly, restore filters: `aws resource-explorer-2 update-view --view-arn <arn> --filters "<original-filters>"`
2. If a view was deleted, recreate it with the original configuration
3. Verify search results match expectations after view restoration
4. Notify dependent teams of any view changes
5. Document the view configuration for future reference

## Common Issues

- symptoms: "Custom view returns fewer results than expected"
  diagnosis: "View filters are excluding resources that match the search query."
  resolution: "Review view filters with get-view. Test with the default view to confirm resources are indexed. Adjust filters."

## Output Format

```yaml
root_cause: "view_configuration — <specific_cause>"
evidence:
  - type: view_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix view filters"
  long_term: "Document view configurations and purposes"
```

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
