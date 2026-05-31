---
title: "B1 — Search Query Errors"
description: "Diagnose search query syntax and execution errors"
status: active
severity: MEDIUM
triggers: ["search error", "query failed", "invalid query", "search syntax error"]
owner: devops-agent
objective: "Resolve search query errors in Resource Explorer"
context: "Resource Explorer uses a specific query syntax. Common errors include wrong field names and incorrect operators."
---

## Phase 1 — Triage

MUST:
- Test the query: `aws resource-explorer-2 search --query-string "<query>" --view-arn <view-arn>`
- Verify the view exists: `aws resource-explorer-2 get-view --view-arn <view-arn>`
- Check index status: `aws resource-explorer-2 get-index`

SHOULD:
- Test with a simple query first: `aws resource-explorer-2 search --query-string "*"`
- Verify the query syntax against documentation

MAY:
- Check supported resource types for the query: `aws resource-explorer-2 list-supported-resource-types`
- Review query examples in AWS documentation

## Phase 2 — Remediate

MUST:
- Fix query syntax errors (field names, operators, quoting)
- Use correct field names: resourcetype, region, tag, tag.key, tag.value, id
- Ensure the view ARN is valid and accessible

SHOULD:
- Start with broad queries and narrow down
- Use resourcetype filter for specific resource searches

MAY:
- Create custom views with pre-configured filters for common queries
- Document working query patterns for the team

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 search: GREEN — read-only resource search"
  - "resource-explorer-2 get-view: GREEN — read-only view details"
  - "resource-explorer-2 get-index: GREEN — read-only index status"
  - "resource-explorer-2 list-supported-resource-types: GREEN — read-only type list"
```

## Escalation Conditions

- Valid queries return errors consistently across multiple views
- Search returns no results despite confirmed indexed resources
- Query performance is unacceptably slow
- Search API returns throttling errors

## Data Sensitivity

- `aws resource-explorer-2 search`: MEDIUM — reveals resource inventory and tags
- `aws resource-explorer-2 get-view`: LOW — reveals view configuration
- Search results: MEDIUM — may reveal resource names, tags, and ARNs

## Prohibited Actions

- NEVER use overly broad queries in production accounts with thousands of resources
- NEVER assume empty search results mean no resources exist
- NEVER share search results without reviewing for sensitive tag values
- NEVER modify views to fix query errors — fix the query syntax instead

## Phase 3 — Rollback

1. No rollback needed for read-only search operations
2. If a view was modified during troubleshooting, restore it: `aws resource-explorer-2 update-view --view-arn <arn> --filters <original-filters>`
3. Document working query patterns for future reference
4. Verify search returns expected results with corrected query
5. Test the corrected query against multiple views

## Common Issues

- symptoms: "Search returns ValidationException with 'Invalid query'"
  diagnosis: "Query syntax error — wrong field name, missing colon, or incorrect quoting."
  resolution: "Use correct syntax: resourcetype:AWS::EC2::Instance, region:us-east-1, tag:Environment, tag.key:Name tag.value:prod"

## Output Format

```yaml
root_cause: "search_query — <specific_cause>"
evidence:
  - type: query_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix query syntax"
  long_term: "Create views with pre-configured filters"
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
