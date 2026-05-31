---
name: costexplorer-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Cost Explorer and
  Budgets problems by analyzing unexpected costs, cost allocation, tagging,
  budget alerts, forecast accuracy, budget actions, anomaly detection,
  savings recommendations, CUR reports, and organizational billing
  following structured runbooks. Activate when: unexpected cost spikes,
  cost allocation issues, untagged resources, budget alerts not firing,
  forecast inaccuracy, budget action failures, anomaly detection problems,
  false positive anomalies, RI/SP recommendation issues, utilization
  tracking, CUR setup problems, report delivery failures, linked account
  cost visibility, consolidated billing issues, or the user says something
  is wrong with Cost Explorer or Budgets without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Cost Explorer, Budgets, Cost and
  Usage Reports, Organizations, IAM, and optionally CloudWatch permissions.
---

# Cost Explorer Diagnostics

## When to use

Any Cost Explorer or Budgets investigation where the console alone is insufficient — unexpected costs, allocation issues, budget alert failures, anomaly detection, savings plan recommendations, CUR configuration, or organizational billing.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ce get-cost-and-usage --time-period Start=<yyyy-mm-dd>,End=<yyyy-mm-dd> --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE
aws budgets describe-budgets --account-id <account-id>
aws ce get-anomalies --date-interval StartDate=<yyyy-mm-dd>,EndDate=<yyyy-mm-dd>
aws ce get-cost-forecast --time-period Start=<yyyy-mm-dd>,End=<yyyy-mm-dd> --metric UNBLENDED_COST --granularity MONTHLY
```

### Step 2 — Domain deep dive

```
aws ce get-cost-and-usage --time-period Start=<yyyy-mm-dd>,End=<yyyy-mm-dd> --granularity DAILY --metrics UnblendedCost --group-by Type=DIMENSION,Key=USAGE_TYPE --filter '{"Dimensions":{"Key":"SERVICE","Values":["<service>"]}}'
aws ce get-tags --time-period Start=<yyyy-mm-dd>,End=<yyyy-mm-dd>
aws ce get-reservation-utilization --time-period Start=<yyyy-mm-dd>,End=<yyyy-mm-dd>
aws ce get-savings-plans-utilization --time-period Start=<yyyy-mm-dd>,End=<yyyy-mm-dd>
aws cur describe-report-definitions
```

Read `references/guardrails.md` before concluding on any Cost Explorer issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ce get-cost-and-usage` | Cost breakdown by service, account, region, tag |
| `ce get-cost-forecast` | Projected costs for future periods |
| `ce get-anomalies` | Detected cost anomalies |
| `ce get-tags` | Available cost allocation tags |
| `ce get-reservation-utilization` | RI utilization and coverage |
| `ce get-savings-plans-utilization` | Savings Plans utilization |
| `budgets describe-budgets` | Budget configuration and status |
| `budgets describe-notifications-for-budget` | Budget alert configuration |
| `cur describe-report-definitions` | CUR report setup |

## Gotchas: Cost Explorer

- Cost Explorer data has a 24-48 hour delay. Do not expect real-time cost visibility.
- Cost allocation tags must be activated in the Billing console. Creating tags on resources does not automatically make them cost allocation tags.
- Budgets evaluate once per day (or per configured period). Budget alerts are not real-time.
- Anomaly detection uses machine learning and needs 2+ weeks of historical data to establish baselines.
- Savings Plans and Reserved Instances show as separate line items. Amortized cost spreads the upfront payment across the term.
- CUR reports are delivered to S3 with a delay of up to 24 hours. The first report may take up to 24 hours to appear.
- Linked account costs in Organizations require the management account to enable Cost Explorer access for member accounts.
- Unblended cost shows the actual charge. Blended cost averages across the organization. Amortized cost spreads upfront payments.
- Cost categories are custom groupings that take up to 24 hours to apply to new data.
- Budget actions can automatically apply SCPs, IAM policies, or target EC2/RDS instances when thresholds are breached.

## Anti-hallucination rules

1. Always cite specific cost data, budget configurations, anomaly details, or CUR report status as evidence.
2. Cost Explorer data is delayed 24-48 hours. Never claim real-time cost visibility.
3. Cost allocation tags must be activated separately. Never assume resource tags are automatically cost allocation tags.
4. Budget alerts evaluate periodically, not in real-time. Never claim instant budget notifications.
5. Anomaly detection needs historical data. Never expect anomaly detection to work on new accounts.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cost Analysis | A1–A3 | Unexpected costs, cost allocation, untagged resources |
| B — Budgets | B1–B3 | Budget alerts, forecast accuracy, action failures |
| C — Anomaly | C1–C2 | Anomaly detection, false positives |
| D — Savings | D1–D2 | RI/SP recommendations, utilization |
| E — Reports | E1–E2 | CUR setup, report delivery |
| F — Organization | F1–F2 | Linked account costs, consolidated billing |
| Z — Catch-All | Z1 | General troubleshooting |
