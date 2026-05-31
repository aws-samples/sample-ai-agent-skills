---
name: cleanrooms-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Clean Rooms
  problems by analyzing collaborations, configured tables, analysis
  rules, and following structured runbooks. Activate when:
  Collaboration creation failures, membership issues, configured table
  errors, analysis rule problems, query failures, output configuration
  issues, differential privacy errors, cryptographic computing problems,
  ML modeling failures, lookalike segment issues, IAM permission errors,
  data access problems, or the user says something is wrong with
  Clean Rooms without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Clean Rooms, Clean Rooms ML,
  S3, Glue, IAM, KMS, CloudTrail, and CloudWatch permissions.
---

# Clean Rooms Diagnostics

## When to use

Any Clean Rooms investigation where the console alone is insufficient — collaboration setup, table configuration, analysis rules, query execution, privacy controls, ML features, or access control issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cleanrooms list-collaborations
aws cleanrooms list-memberships
aws cleanrooms get-collaboration --collaboration-identifier <id>
aws cleanrooms get-membership --membership-identifier <id>
aws cleanrooms list-configured-tables
```

### Step 2 — Deep dive

```
aws cleanrooms get-configured-table --configured-table-identifier <id>
aws cleanrooms get-configured-table-analysis-rule --configured-table-identifier <id> --analysis-rule-type <AGGREGATION|LIST|CUSTOM>
aws cleanrooms list-configured-table-associations --membership-identifier <id>
aws cleanrooms list-protected-queries --membership-identifier <id>
aws cleanrooms get-protected-query --membership-identifier <id> --protected-query-identifier <query-id>
aws cleanrooms get-collaboration-privacy-budget-template --collaboration-identifier <id> --privacy-budget-template-identifier <template-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cleanrooms.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Clean Rooms issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `cleanrooms list-collaborations` | List all collaborations |
| `cleanrooms get-collaboration` | Collaboration details and status |
| `cleanrooms list-memberships` | List memberships |
| `cleanrooms get-membership` | Membership details and abilities |
| `cleanrooms list-configured-tables` | List configured tables |
| `cleanrooms get-configured-table` | Table configuration details |
| `cleanrooms get-configured-table-analysis-rule` | Analysis rule details |
| `cleanrooms list-protected-queries` | List queries in a membership |
| `cleanrooms get-protected-query` | Query details and results |
| `cleanrooms list-schemas` | List schemas in a collaboration |
| `cleanrooms get-schema` | Schema details |
| `cleanrooms-ml list-training-datasets` | List ML training datasets |

## Gotchas: Clean Rooms

- Collaborations require at least two members: A collaboration must have at least two member accounts. A single account cannot create a collaboration with itself for testing — use two separate AWS accounts.
- Analysis rules restrict query capabilities: Each configured table must have an analysis rule (AGGREGATION, LIST, or CUSTOM) that defines what queries are allowed. Without an analysis rule, the table cannot be queried.
- Configured tables reference Glue Data Catalog: Tables must be registered in the Glue Data Catalog before they can be configured in Clean Rooms. Missing Glue tables cause configuration failures.
- Query results go to the member who can receive results: Only members with the CAN_RECEIVE_RESULTS ability can receive query output. The output S3 bucket must be in the same account as the receiving member.
- Differential privacy has a budget: Differential privacy uses a privacy budget that depletes with each query. Once exhausted, no more queries can run until the budget is refreshed. Monitor budget consumption.
- Cryptographic computing requires specific setup: Clean Rooms cryptographic computing requires client-side encryption of data before upload. It uses specific encryption schemas and cannot be enabled retroactively on existing tables.
- ML features are separate from core Clean Rooms: Clean Rooms ML (lookalike modeling, etc.) uses a separate API (`cleanrooms-ml`) and has its own permissions and configuration requirements.
- Cross-account data never leaves the owner's account: Clean Rooms processes queries without moving raw data between accounts. Only aggregated or filtered results are shared. This is a fundamental design principle.

## Anti-hallucination rules

1. Always cite specific collaboration IDs, membership IDs, or API responses as evidence.
2. Collaborations need at least two members — never suggest single-account collaborations.
3. Analysis rules are required — never claim tables can be queried without them.
4. Raw data never leaves the owner's account — never suggest data is copied between accounts.
5. Differential privacy budget depletes — never claim unlimited queries with differential privacy.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 18 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Collaboration | A1-A2 | Creation, membership |
| B — Tables | B1-B2 | Configured tables, analysis rules |
| C — Queries | C1-C2 | Query failures, output configuration |
| D — Privacy | D1-D2 | Differential privacy, cryptographic computing |
| E — ML | E1-E2 | ML modeling, lookalike segments |
| F — Security | F1-F2 | IAM permissions, data access |
| Z — Catch-All | Z1 | General Clean Rooms troubleshooting |
