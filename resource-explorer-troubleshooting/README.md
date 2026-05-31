# AWS Resource Explorer Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Resource Explorer problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Resource Explorer — index creation, aggregator configuration, search queries, missing resources, views, filter expressions, cross-region indexing, IAM permissions, organization integration, and supported resource types.

### Activate When

- Index creation failures
- Aggregator index issues
- Search query errors
- Resources missing from search results
- View configuration problems
- Filter expression errors
- Cross-region indexing failures
- IAM permission errors
- Organization integration issues
- Supported resource type questions
- General Resource Explorer errors

---

## Skill Structure

```
resource-explorer-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-index-creation-failures.md
    ├── A2-aggregator-index-issues.md
    ├── B1-search-query-errors.md
    ├── B2-missing-resources.md
    ├── C1-view-configuration.md
    ├── C2-filter-expression-errors.md
    ├── D1-cross-region-indexing.md
    ├── D2-index-type-issues.md
    ├── E1-iam-permissions.md
    ├── E2-organization-integration.md
    ├── F1-supported-resource-types.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Indexes** | A1–A2 | Creation, aggregator |
| **B — Search** | B1–B2 | Query errors, missing resources |
| **C — Views** | C1–C2 | Configuration, filters |
| **D — Cross-Region** | D1–D2 | Indexing, index types |
| **E — Permissions** | E1–E2 | IAM, organization integration |
| **F — Resources** | F1 | Supported resource types |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `resource-explorer-2:*`, `iam:Get*`, `organizations:Describe*` permissions

---

## License

MIT-0
