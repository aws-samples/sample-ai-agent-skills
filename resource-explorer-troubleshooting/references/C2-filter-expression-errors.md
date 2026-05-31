---
title: "C2 — Filter Expression Errors"
description: "Diagnose filter expression syntax errors in Resource Explorer"
status: active
severity: MEDIUM
triggers: ["filter error", "invalid filter", "filter syntax", "expression error"]
owner: devops-agent
objective: "Resolve filter expression syntax and logic errors"
context: "Resource Explorer uses a specific query syntax. Filter expressions in views and search queries have different contexts."
---

## Phase 1 — Triage

MUST:
- Test the filter expression: `aws resource-explorer-2 search --query-string "<filter>" --view-arn <view-arn>`
- Verify field names are correct (resourcetype, region, tag, tag.key, tag.value, id)
- Check for syntax errors: missing colons, incorrect quoting, invalid operators

SHOULD:
- Test with a known-working filter to verify the search infrastructure
- Compare the filter against documented syntax examples

MAY:
- Review AWS documentation for filter expression syntax
- Test individual filter components separately

## Phase 2 — Remediate

MUST:
- Fix field name errors (use lowercase: resourcetype not ResourceType)
- Correct syntax: use colons between field and value (resourcetype:AWS::EC2::Instance)
- Remove unsupported operators or functions

SHOULD:
- Build complex filters incrementally, testing each component
- Use tag.key and tag.value for tag-based filtering

MAY:
- Create views with pre-built filters for common search patterns
- Document filter syntax reference for the team

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 search: GREEN — read-only search operation"
  - "resource-explorer-2 update-view: YELLOW — modifies view filter configuration"
  - "filter testing: GREEN — read-only query execution"
```

## Escalation Conditions

- Valid filter expressions return unexpected results consistently
- Filter syntax documentation is unclear or contradictory
- Complex filter requirements cannot be expressed in the query language
- Filter performance is unacceptable for large resource inventories

## Data Sensitivity

- Search results: MEDIUM — reveals resource inventory and tags
- Filter expressions: LOW — reveals search criteria
- Tag values in filters: MEDIUM — may contain environment or ownership information

## Prohibited Actions

- NEVER use overly broad filters that return all resources in large accounts
- NEVER assume filter syntax is the same as other AWS query languages (CloudWatch, Athena)
- NEVER modify view filters in production without testing first
- NEVER share filter results without reviewing for sensitive tag values

## Phase 3 — Rollback

1. If view filters were modified, restore original: `aws resource-explorer-2 update-view --view-arn <arn> --filters "<original>"`
2. Document the working filter syntax for future reference
3. Verify search results are correct after filter changes
4. Test the corrected filter against multiple resource types
5. Update team documentation with filter syntax examples

## Common Issues

- symptoms: "Filter 'ResourceType:AWS::EC2::Instance' returns no results"
  diagnosis: "Field names are case-sensitive and must be lowercase."
  resolution: "Use lowercase: resourcetype:AWS::EC2::Instance (not ResourceType)."

## Output Format

```yaml
root_cause: "filter_expression — <specific_cause>"
evidence:
  - type: filter_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter syntax"
  long_term: "Create views with pre-built filters"
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
