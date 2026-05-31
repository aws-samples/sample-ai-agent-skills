---
name: resource-explorer-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Resource Explorer
  problems by analyzing index creation, aggregator indexes, search queries,
  missing resources, views, filters, cross-region indexing, IAM permissions,
  organization integration, and supported resource types.
  Activate when: index creation failures, search errors, missing resources,
  view configuration issues, cross-region problems, permission errors,
  or the user says something is wrong with Resource Explorer.
compatibility: >
  Requires AWS CLI or SDK access with resource-explorer-2, iam,
  organizations, and cloudwatch permissions.
---

# AWS Resource Explorer Diagnostics

## When to use

Any AWS Resource Explorer investigation — index creation, aggregator configuration, search queries, missing resources, view management, filter expressions, cross-region indexing, IAM permissions, organization integration, or supported resource types.

## Investigation workflow

### Step 1 — Collect and triage

```
aws resource-explorer-2 list-indexes --query 'Indexes[*].{Region:Region,Type:Type,Arn:Arn}'
aws resource-explorer-2 get-index
aws resource-explorer-2 list-views --query 'Views[*]'
```

### Step 2 — Domain deep dive

```
aws resource-explorer-2 search --query-string "resourcetype:AWS::EC2::Instance" --view-arn <view-arn>
aws resource-explorer-2 get-view --view-arn <view-arn>
aws resource-explorer-2 list-supported-resource-types --query 'ResourceTypes[*].{Service:Service,ResourceType:ResourceType}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=resource-explorer-2.amazonaws.com --max-results 20
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names resource-explorer-2:Search resource-explorer-2:GetIndex
aws resource-explorer-2 list-indexes --type AGGREGATOR
```

Read `references/guardrails.md` before concluding on any Resource Explorer issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `resource-explorer-2 list-indexes` | List all indexes across regions |
| `resource-explorer-2 get-index` | Get index details for current region |
| `resource-explorer-2 search` | Search for resources |
| `resource-explorer-2 list-views` | List configured views |
| `resource-explorer-2 get-view` | Get view details and filters |
| `resource-explorer-2 list-supported-resource-types` | List supported resource types |
| `resource-explorer-2 create-index` | Create a local or aggregator index |

## Gotchas: AWS Resource Explorer

- Resource Explorer requires an index in EACH region you want to search. Without a local index, resources in that region are invisible to search.
- Only ONE aggregator index is allowed per account. The aggregator collects data from all local indexes. Attempting to create a second aggregator fails.
- Search results may be delayed up to 36 hours after resource creation. Resource Explorer does not provide real-time resource discovery.
- Views filter search results. The default view shows all resources, but custom views with filters may hide resources the user expects to see.
- Filter expressions use a specific syntax. Common mistakes include wrong field names, missing quotes, and incorrect operators.
- Not all AWS resource types are supported. Resource Explorer indexes a subset of AWS resource types. Unsupported types will never appear in search results.
- Cross-region search requires both a local index in the source region AND an aggregator index. Without both, cross-region queries return incomplete results.

## Anti-hallucination rules

1. Always cite specific index ARNs, view ARNs, or search query results as evidence.
2. Local indexes and aggregator indexes serve different purposes. Never conflate them.
3. Search delays are normal (up to 36 hours). Never claim Resource Explorer provides real-time discovery.
4. Views filter results. Always check the view configuration before concluding resources are missing.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Indexes | A1–A2 | Index creation, aggregator issues |
| B — Search | B1–B2 | Query errors, missing resources |
| C — Views | C1–C2 | View configuration, filter expressions |
| D — Cross-Region | D1–D2 | Cross-region indexing, index types |
| E — Permissions | E1–E2 | IAM permissions, organization integration |
| F — Resources | F1 | Supported resource types |
| Z — Catch-All | Z1 | General troubleshooting |
