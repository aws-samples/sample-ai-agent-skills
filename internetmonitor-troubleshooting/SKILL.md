---
name: internetmonitor-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CloudWatch Internet Monitor
  problems by analyzing monitor creation, configuration, health events, availability
  scores, traffic insights, city-network analysis, CloudFront/VPC integrations,
  log delivery, threshold configuration, and following structured runbooks.
  Activate when: monitor creation failures, health event analysis issues, availability
  score problems, traffic insight errors, integration failures, log delivery issues,
  threshold configuration problems, or the user says something is wrong with Internet Monitor.
compatibility: >
  Requires AWS CLI or SDK access with internetmonitor, cloudwatch, cloudfront,
  ec2, s3, logs, and iam permissions.
---

# Amazon CloudWatch Internet Monitor Diagnostics

## When to use

Any CloudWatch Internet Monitor investigation — monitor creation, configuration, health event analysis, availability score issues, traffic insights, city-network analysis, CloudFront/VPC integrations, log delivery, or threshold configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws internetmonitor list-monitors --query 'Monitors[*].{Name:MonitorName,Status:Status,ProcessingStatus:ProcessingStatus}'
aws internetmonitor get-monitor --monitor-name <name> --query '{Status:Status,Resources:Resources,MaxCityNetworksToMonitor:MaxCityNetworksToMonitor}'
aws internetmonitor list-health-events --monitor-name <name> --event-status ACTIVE
```

### Step 2 — Domain deep dive

```
aws internetmonitor get-health-event --monitor-name <name> --event-id <event-id>
aws internetmonitor get-internet-event --event-id <event-id>
aws internetmonitor get-query-results --monitor-name <name> --query-id <query-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=internetmonitor.amazonaws.com --max-results 20
aws internetmonitor list-internet-events --query 'InternetEvents[*].{EventId:EventId,Status:Status,StartedAt:StartedAt}'
aws logs describe-log-groups --log-group-name-prefix /aws/internetmonitor
```

Read `references/guardrails.md` before concluding on any Internet Monitor issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `internetmonitor list-monitors` | List all monitors |
| `internetmonitor get-monitor` | Get monitor details and configuration |
| `internetmonitor list-health-events` | List health events for a monitor |
| `internetmonitor get-health-event` | Get health event details |
| `internetmonitor get-internet-event` | Get global internet event details |
| `internetmonitor start-query` | Start a data query for traffic insights |
| `internetmonitor get-query-results` | Retrieve query results |

## Gotchas: CloudWatch Internet Monitor

- Internet Monitor requires at least one monitored resource (CloudFront distribution or VPC). A monitor with no resources will show INACTIVE status.
- MaxCityNetworksToMonitor controls how many city-networks are tracked. The default is 500,000. Higher values increase cost but improve coverage.
- Health events are generated when performance or availability drops below thresholds. Default thresholds are availability < 96% and performance < 95%.
- Internet Monitor uses passive traffic analysis — it does NOT send synthetic probes. It analyzes existing application traffic patterns.
- Availability scores and performance scores are calculated per city-network pair. A single city-network issue may not affect the overall score significantly.
- S3 and CloudWatch Logs delivery must be configured explicitly. They are not enabled by default.
- Internet Monitor is regional. Each monitor operates in the region where it is created and monitors resources in that region.

## Anti-hallucination rules

1. Always cite specific monitor names, event IDs, or API responses as evidence.
2. Internet Monitor uses passive traffic analysis, NOT synthetic monitoring. Never claim it sends probes.
3. Health events reflect observed degradation, not predicted issues. Never claim predictive capability.
4. Availability and performance scores are per city-network. Never present a single city-network score as the overall score.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Monitor Setup | A1–A2 | Creation failures, configuration |
| B — Health Events | B1–B2 | Health event analysis, availability score issues |
| C — Traffic Insights | C1–C2 | Traffic insights errors, city-network analysis |
| D — Integrations | D1–D2 | CloudFront integration, VPC integration |
| E — Log Delivery | E1–E2 | S3 log delivery, CloudWatch log delivery |
| F — Thresholds | F1 | Threshold configuration |
| Z — Catch-All | Z1 | General troubleshooting |
