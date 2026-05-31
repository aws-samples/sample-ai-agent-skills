---
name: monitron-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Monitron
  problems by analyzing project creation, gateway setup, sensor pairing,
  data gaps, inference anomalies, false positives, gateway connectivity,
  WiFi configuration, user management, mobile app issues, data export,
  and following structured runbooks.
  Activate when: project creation failures, gateway setup issues,
  sensor pairing errors, sensor data gaps, inference anomalies,
  false positive alerts, gateway connectivity problems, WiFi
  configuration issues, user management errors, mobile app issues,
  data export problems, or the user says something is wrong with Monitron.
compatibility: >
  Requires AWS CLI or SDK access with monitron, s3, iam, kms,
  cloudtrail, and cloudwatch permissions. Physical access to
  Monitron gateways and sensors may be required.
---

# Amazon Monitron Diagnostics

## When to use

Any Amazon Monitron investigation — project creation, gateway setup, sensor pairing, data gaps, inference anomalies, false positives, gateway connectivity, WiFi configuration, user management, mobile app issues, or data export.

## Investigation workflow

### Step 1 — Collect and triage

```
aws monitron list-projects --query 'projectSummaries[*].{Name:name,Arn:projectArn,CreatedAt:createdAt}'
aws monitron get-project --project-id <project-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=monitron.amazonaws.com --max-results 20
```

### Step 2 — Domain deep dive

```
aws monitron list-project-admin-users --project-id <project-id>
aws s3 ls s3://<monitron-export-bucket>/<prefix>/
aws iam get-role --role-name <monitron-service-role>
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/Monitron --metric-name SensorDataPoints --dimensions Name=ProjectId,Value=<project-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws kms describe-key --key-id <monitron-kms-key-id>
aws sso-admin list-account-assignments --instance-arn <sso-instance-arn> --account-id <account-id> --permission-set-arn <ps-arn>
```

Read `references/guardrails.md` before concluding on any Monitron issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `monitron list-projects` | List Monitron projects |
| `monitron get-project` | Get project details |
| `monitron list-project-admin-users` | List project admins |
| `monitron tag-resource` | Tag Monitron resources |
| `cloudwatch get-metric-statistics` | Check sensor data metrics |
| `s3 ls` | Check data export output |
| `iam get-role` | Verify service role |

## Gotchas: Amazon Monitron

- Monitron uses dedicated hardware (gateways and sensors). Software-only troubleshooting may not resolve hardware issues.
- Gateways require WiFi connectivity with specific requirements: 2.4 GHz band, WPA2 security, and internet access to AWS endpoints.
- Sensors communicate with gateways via Bluetooth Low Energy (BLE). Range is limited to approximately 20-30 meters with line of sight.
- Monitron uses machine learning inference that requires a learning period. New sensors need 14+ days of data before anomaly detection is reliable.
- The Monitron mobile app is required for initial gateway and sensor setup. CLI/API cannot perform physical device pairing.
- Data export to S3 is asynchronous and may have delays. Export format is specific to Monitron and requires parsing.
- Monitron projects are regional. Gateways and sensors are bound to the project region.

## Anti-hallucination rules

1. Always cite specific project IDs, gateway IDs, or API responses as evidence.
2. Hardware issues (gateway, sensor) cannot be resolved through API calls alone. Acknowledge physical intervention needs.
3. The 14-day learning period is mandatory. Never claim anomaly detection works immediately.
4. Monitron mobile app is required for device setup. Never suggest CLI-only device pairing.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Project & Gateway | A1–A2 | Project creation failures, gateway setup issues |
| B — Sensors | B1–B2 | Sensor pairing errors, sensor data gaps |
| C — Inference | C1–C2 | Inference anomalies, false positive alerts |
| D — Connectivity | D1–D2 | Gateway connectivity, WiFi configuration |
| E — Users | E1–E2 | User management, mobile app issues |
| F — Data | F1 | Data export |
| Z — Catch-All | Z1 | General troubleshooting |
