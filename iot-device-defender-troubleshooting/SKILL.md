---
name: iot-device-defender-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT Device
  Defender problems by analyzing audit configuration, audit findings,
  security profiles, metric anomalies, mitigation actions, custom
  metrics, alert targets, SNS notifications, ML Detect, dimensions,
  fleet indexing integration, and following structured runbooks.
  Activate when: audit configuration errors, security profile issues,
  metric anomalies, mitigation action failures, alert delivery
  problems, or the user says something is wrong with IoT Device
  Defender.
compatibility: >
  Requires AWS CLI or SDK access with iot, iot-device-defender,
  sns, iam, cloudwatch, and cloudtrail permissions.
---

# AWS IoT Device Defender Diagnostics

## When to use

Any AWS IoT Device Defender investigation — audit configuration, audit findings, security profiles, metric anomalies, mitigation actions, custom metrics, alert targets, SNS notifications, ML Detect, dimensions, or fleet indexing integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iot describe-account-audit-configuration
aws iot list-audit-tasks --task-status COMPLETED --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S)
aws iot list-security-profiles --query 'securityProfileIdentifiers[*].{Name:name,Arn:arn}'
```

### Step 2 — Domain deep dive

```
aws iot list-audit-findings --task-id <task-id> --query 'findings[*].{CheckName:checkName,Severity:severity,NonCompliantResource:nonCompliantResource}'
aws iot describe-security-profile --security-profile-name <profile-name>
aws iot list-active-violations --security-profile-name <profile-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iot.amazonaws.com --max-results 20
aws iot list-mitigation-actions --query 'actionIdentifiers[*].{Name:actionName,Arn:actionArn}'
aws iot describe-audit-mitigation-actions-task --task-id <task-id>
```

Read `references/guardrails.md` before concluding on any IoT Device Defender issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `iot describe-account-audit-configuration` | Check audit setup |
| `iot list-audit-tasks` | List audit executions |
| `iot list-audit-findings` | Get audit findings |
| `iot list-security-profiles` | List Detect profiles |
| `iot list-active-violations` | Check active violations |
| `iot list-mitigation-actions` | List mitigation actions |
| `iot describe-security-profile` | Get profile details |

## Gotchas: IoT Device Defender

- Device Defender has TWO main features: Audit (checks IoT configuration against best practices) and Detect (monitors device behavior for anomalies). They are configured separately.
- Audit checks run on a schedule or on-demand. They check account-level IoT configuration, not individual device behavior.
- Detect uses security profiles attached to thing groups or all things. Profiles define expected behaviors and thresholds.
- ML Detect requires a 14-day training period before it can detect anomalies. During training, no violations are generated.
- Custom metrics must be reported by devices using the MQTT reserved topic $aws/things/{thingName}/defender/metrics/json.
- Mitigation actions are pre-defined actions (e.g., add thing to group, enable logging) that can be applied to audit findings.
- Alert targets (SNS topics) must be configured per security profile. Without alert targets, violations are recorded but no notifications are sent.

## Anti-hallucination rules

1. Always cite specific audit task IDs, security profile names, or violation IDs as evidence.
2. Audit and Detect are separate features with different configurations. Never conflate them.
3. ML Detect requires a 14-day training period. Never claim it works immediately.
4. Custom metrics require device-side reporting. Never assume metrics are collected automatically.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Audit | A1-A2 | Audit configuration, finding analysis |
| B — Detect | B1-B2 | Security profiles, metric anomalies |
| C — Mitigation | C1-C2 | Mitigation actions, custom metrics |
| D — Alerts | D1-D2 | Alert targets, SNS notifications |
| E — ML & Dimensions | E1-E2 | ML Detect, dimension configuration |
| F — Integration | F1 | Fleet indexing integration |
| Z — Catch-All | Z1 | General troubleshooting |
