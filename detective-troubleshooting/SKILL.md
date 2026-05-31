---
name: detective-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Detective
  problems by analyzing behavior graphs, data sources, investigation
  workflows, and following structured runbooks. Activate when:
  Behavior graph creation failures, data source enablement issues,
  finding investigation problems, entity profile errors, anomaly
  detection issues, admin account configuration, member invitation
  failures, GuardDuty integration problems, Security Hub integration
  issues, data volume concerns, retention questions, IAM permission
  errors, cross-account access issues, or the user says something is
  wrong with Detective without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Detective, GuardDuty, Security Hub,
  IAM, CloudTrail, and CloudWatch permissions.
---

# Detective Diagnostics

## When to use

Any Amazon Detective investigation where the console alone is insufficient — behavior graph issues, data source problems, finding investigation failures, multi-account management, integration with GuardDuty/Security Hub, data volume concerns, or access control issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws detective list-graphs
aws detective list-members --graph-arn <graph-arn>
aws detective list-datasource-packages --graph-arn <graph-arn>
aws detective get-members --graph-arn <graph-arn> --account-ids <account-id>
aws detective list-invitations
```

### Step 2 — Deep dive

```
aws detective list-investigations --graph-arn <graph-arn>
aws detective get-investigation --graph-arn <graph-arn> --investigation-id <id>
aws detective list-indicators --graph-arn <graph-arn> --investigation-id <id>
aws detective batch-get-graph-member-datasources --graph-arn <graph-arn> --account-ids <account-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=detective.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Detective issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `detective list-graphs` | List behavior graphs |
| `detective list-members` | List member accounts in a graph |
| `detective get-members` | Get member account details and status |
| `detective list-datasource-packages` | Check enabled data sources |
| `detective list-invitations` | Check pending invitations |
| `detective list-investigations` | List automated investigations |
| `detective get-investigation` | Get investigation details |
| `detective list-indicators` | Get investigation indicators |
| `detective batch-get-graph-member-datasources` | Data source status per member |
| `detective batch-get-membership-datasources` | Data source status for current account |

## Gotchas: Detective

- Detective requires GuardDuty enabled first: Detective uses GuardDuty findings as a primary data source. GuardDuty must be enabled for at least 48 hours before Detective can be enabled.
- Behavior graph has a 1200-account limit: A single behavior graph supports up to 1200 member accounts. For larger organizations, use multiple graphs.
- Data ingestion takes time: After enabling Detective or adding a member, it takes up to 24 hours for data to appear in the behavior graph. Historical data is backfilled for up to 12 months.
- Detective is regional: Each region has its own behavior graph. There is no cross-region aggregation. Enable Detective in each region where you need investigation capabilities.
- Admin account cannot be changed easily: The delegated administrator account for Detective cannot be changed without disabling and re-enabling the service. Plan the admin account carefully.
- Free trial is 30 days: Detective offers a 30-day free trial. After that, charges are based on data volume ingested. Monitor costs during the trial period.
- Automated investigations require specific data sources: Automated investigation features require GuardDuty and CloudTrail data sources to be enabled. Missing data sources limit investigation capabilities.
- Member accounts must accept invitations: Unlike some services, Detective member accounts must explicitly accept invitations (unless using Organizations integration with auto-enable).

## Anti-hallucination rules

1. Always cite specific graph ARNs, investigation IDs, or API responses as evidence.
2. Detective requires GuardDuty — never suggest enabling Detective without GuardDuty.
3. Data takes up to 24 hours to appear — never claim instant data availability.
4. Detective is regional — never claim cross-region behavior graph aggregation.
5. Behavior graph limit is 1200 accounts — never suggest unlimited member accounts.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 18 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Graph | A1-A2 | Behavior graph issues, data sources |
| B — Investigation | B1-B3 | Finding investigation, entity profiles, anomalies |
| C — Multi-Account | C1-C2 | Admin account, member invitations |
| D — Integration | D1-D2 | GuardDuty findings, Security Hub |
| E — Data | E1-E2 | Data volume, retention |
| F — Access | F1-F2 | IAM permissions, cross-account |
| Z — Catch-All | Z1 | General Detective troubleshooting |
