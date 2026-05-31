---
name: cloudwatch-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CloudWatch problems
  by analyzing log groups, metrics, alarms, Synthetics canaries, Contributor
  Insights rules, and cross-account observability configurations following
  structured runbooks. Activate when: missing logs, log group retention
  issues, subscription filter failures, missing or delayed metrics, custom
  metric publishing errors, metric math problems, INSUFFICIENT_DATA alarms,
  alarms not triggering, composite alarm misconfiguration, alarm action
  failures, Synthetics canary failures, canary VPC issues, canary script
  errors, Log Insights query errors or timeouts, Contributor Insights rule
  or cost issues, cross-account observability access problems, CloudWatch
  Agent issues, or the user says something is wrong with CloudWatch without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with CloudWatch, CloudWatch Logs,
  CloudWatch Synthetics, IAM, Lambda, and optionally EC2/ECS/EKS
  permissions.
---

# CloudWatch Diagnostics

## When to use

Any CloudWatch investigation where the console alone is insufficient — missing logs, metric gaps, alarm misfires, Synthetics canary failures, Log Insights query problems, Contributor Insights cost surprises, cross-account observability setup, or CloudWatch Agent configuration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws logs describe-log-groups --log-group-name-prefix <prefix>
aws logs filter-log-events --log-group-name <name> --start-time <epoch-ms> --end-time <epoch-ms>
aws cloudwatch describe-alarms --alarm-names <name>
aws cloudwatch get-metric-statistics --namespace <ns> --metric-name <metric> --dimensions Name=<key>,Value=<val> --start-time <iso> --end-time <iso> --period <sec> --statistics <stat>
aws synthetics describe-canaries --names <canary-name>
aws cloudwatch describe-anomaly-detectors
```

### Step 2 — Domain deep dive

```
aws logs describe-subscription-filters --log-group-name <name>
aws logs describe-metric-filters --log-group-name <name>
aws cloudwatch describe-alarms --state-value INSUFFICIENT_DATA
aws cloudwatch get-metric-data --metric-data-queries file://queries.json --start-time <iso> --end-time <iso>
aws synthetics get-canary-runs --name <canary-name>
aws cloudwatch describe-insight-rules
aws oam list-links
aws oam list-sinks
```

Read `references/cloudwatch-guardrails.md` before concluding on any CloudWatch issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `logs describe-log-groups` | Log group config, retention, KMS, stored bytes |
| `logs filter-log-events` | Search log events by pattern and time range |
| `logs describe-subscription-filters` | Subscription filter destinations and patterns |
| `logs describe-metric-filters` | Metric filters extracting metrics from logs |
| `cloudwatch describe-alarms` | Alarm config, state, actions, evaluation |
| `cloudwatch get-metric-statistics` | Raw metric datapoints for a time range |
| `cloudwatch get-metric-data` | Metric math, multiple metrics, complex queries |
| `synthetics describe-canaries` | Canary config, runtime, schedule, VPC |
| `synthetics get-canary-runs` | Canary execution history and failure details |
| `cloudwatch describe-insight-rules` | Contributor Insights rule definitions |
| `oam list-links` / `oam list-sinks` | Cross-account observability configuration |

## Gotchas: CloudWatch

- Log group retention is infinite by default. This means logs are stored forever and costs accumulate. Always check and set retention policies explicitly.
- Metric resolution: standard metrics have 1-minute resolution. High-resolution metrics have 1-second resolution but cost more and are only available for custom metrics.
- Alarm evaluation periods vs datapoints to alarm: an alarm can evaluate over N periods but only require M of those N to breach. Misunderstanding this causes alarms that seem to not trigger.
- INSUFFICIENT_DATA means no data was received for the metric during the evaluation period. It does not necessarily indicate a problem — the resource may simply not be emitting data.
- Composite alarms aggregate other alarms but do not have their own actions by default. Actions must be configured separately on the composite alarm.
- Metric math expressions cannot trigger alarms on MISSING data. If any input metric has missing data, the expression result is also missing.
- Cross-account observability requires CloudWatch Observability Access Manager (OAM) with a sink in the monitoring account and links from source accounts.
- Synthetics canaries run on Lambda under the hood. They inherit Lambda limits (15-minute timeout, memory, /tmp storage, VPC cold starts).
- Contributor Insights rules have cost per matched log event. High-volume log groups can generate unexpected charges.
- CloudWatch Logs Insights queries have a 15-minute timeout. Complex queries on large log groups may need time range narrowing or query optimization.

### Metric resolution comparison

| Resolution | Period | Retention | Use Case |
|------------|--------|-----------|----------|
| Standard | 1 minute | 15 days (1-min), 63 days (5-min), 455 days (1-hr) | AWS service metrics |
| High-resolution | 1 second | 3 hours (1-sec), 15 days (1-min), 63 days (5-min), 455 days (1-hr) | Custom metrics needing sub-minute granularity |

### Alarm evaluation logic

| Setting | Meaning |
|---------|---------|
| Period | Length of each evaluation window |
| Evaluation Periods | Number of consecutive periods to consider |
| Datapoints to Alarm | How many of those periods must breach threshold |
| Treat Missing Data | What to do when no data: missing, notBreaching, breaching, ignore |

## Anti-hallucination rules

1. Always cite specific log group configurations, metric datapoints, alarm state history, or canary run results as evidence.
2. INSUFFICIENT_DATA is not an error state. Never claim it means the monitored resource is down without checking metric publishing.
3. Log group retention defaults to Never Expire. Never assume logs are automatically deleted.
4. Composite alarms do not inherit actions from child alarms. Never claim actions propagate automatically.
5. Metric math with missing data returns missing. Never claim metric math handles missing data gracefully.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Logs | A1-A4 | Missing logs, log group retention, subscription filters, cross-account log delivery |
| B — Metrics | B1-B3 | Missing metrics, custom metric publishing, metric math errors |
| C — Alarms | C1-C4 | INSUFFICIENT_DATA, alarm not triggering, composite alarms, alarm actions |
| D — Synthetics | D1-D3 | Canary failures, canary VPC issues, canary script errors |
| E — Log Insights | E1-E2 | Query errors, query performance |
| F — Contributor Insights | F1-F2 | Rule configuration, high cost |
| G — Cross-Account | G1-G2 | Observability access manager, cross-account metrics/logs |
| H — Agent | H1-H2 | CloudWatch Agent issues, unified agent configuration |
| Z — Catch-All | Z1 | General troubleshooting |
