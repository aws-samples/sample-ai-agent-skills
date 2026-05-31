---
name: rum-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CloudWatch RUM
  problems by analyzing app monitor creation, configuration, data collection,
  JavaScript snippet issues, custom events, page performance, session replay,
  user journey analysis, Cognito integration, CloudWatch metric export,
  data retention, and following structured runbooks.
  Activate when: app monitor creation failures, data collection issues,
  JavaScript snippet problems, custom event errors, performance analysis
  issues, session replay failures, Cognito integration problems, metric
  export issues, or the user says something is wrong with CloudWatch RUM.
compatibility: >
  Requires AWS CLI or SDK access with rum, cloudwatch, cognito-identity,
  cognito-idp, s3, logs, and iam permissions.
---

# Amazon CloudWatch RUM Diagnostics

## When to use

Any CloudWatch RUM investigation — app monitor creation, configuration, data collection, JavaScript snippet issues, custom events, page performance analysis, session replay, user journey analysis, Cognito integration, CloudWatch metric export, or data retention configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rum list-app-monitors --query 'AppMonitorSummaries[*].{Name:Name,State:State,Id:Id}'
aws rum get-app-monitor --name <name> --query '{Name:Name,State:State,Domain:AppMonitorConfiguration.SessionSampleRate}'
aws rum get-app-monitor-data --name <name> --time-range '{"After":<epoch-start>,"Before":<epoch-end>}' --max-results 10
```

### Step 2 — Domain deep dive

```
aws rum get-app-monitor --name <name>
aws rum list-rum-metrics-destinations --app-monitor-name <name>
aws rum batch-get-rum-metric-definitions --app-monitor-name <name> --destination CloudWatch
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rum.amazonaws.com --max-results 20
aws cognito-identity describe-identity-pool --identity-pool-id <pool-id>
aws cloudwatch list-metrics --namespace AWS/RUM --dimensions Name=AppMonitorName,Value=<name>
```

Read `references/guardrails.md` before concluding on any RUM issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `rum list-app-monitors` | List all app monitors |
| `rum get-app-monitor` | Get app monitor details and configuration |
| `rum get-app-monitor-data` | Retrieve collected RUM data |
| `rum list-rum-metrics-destinations` | List metric export destinations |
| `rum batch-get-rum-metric-definitions` | Get metric definitions for export |
| `rum put-rum-events` | Send RUM events (used by the web client) |
| `rum create-app-monitor` | Create a new app monitor |

## Gotchas: CloudWatch RUM

- RUM requires a JavaScript snippet embedded in the web application. Without the snippet, no data is collected. The snippet must be loaded before other scripts.
- RUM uses Amazon Cognito Identity pools for unauthenticated access. The identity pool must allow unauthenticated identities and have the correct IAM role.
- Session sample rate controls what percentage of sessions are recorded. A rate of 1.0 means 100%. Lower rates reduce cost but provide less data.
- Custom events require explicit configuration in the app monitor. They are not collected by default.
- Session replay captures DOM changes and user interactions. It increases data volume and cost significantly.
- CloudWatch metric export must be explicitly configured. RUM data is not automatically available as CloudWatch metrics.
- RUM is regional. Each app monitor operates in the region where it is created. The JavaScript snippet sends data to that region.

## Anti-hallucination rules

1. Always cite specific app monitor names, configuration details, or API responses as evidence.
2. RUM collects data from real user browsers, NOT synthetic tests. Never claim RUM sends synthetic probes.
3. The JavaScript snippet is required for data collection. Never assume data collection works without verifying snippet installation.
4. Session replay is optional and must be enabled. Never assume session replay is active by default.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — App Monitor Setup | A1–A2 | Creation, configuration |
| B — Data Collection | B1–B2 | Collection failures, JavaScript snippet issues |
| C — Events & Performance | C1–C2 | Custom event errors, page performance analysis |
| D — Session Analysis | D1–D2 | Session replay issues, user journey analysis |
| E — Integrations | E1–E2 | Cognito integration, CloudWatch metric export |
| F — Data Management | F1 | Data retention configuration |
| Z — Catch-All | Z1 | General troubleshooting |
