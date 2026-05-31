---
name: devopsguru-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon DevOps Guru problems
  by analyzing insights, anomaly detection, resource coverage, notification
  channels, cost estimation, proactive insights, and following structured
  runbooks. Activate when: missing insights, false anomalies, resource
  coverage gaps, notification failures, cost estimation issues, or the
  user says something is wrong with DevOps Guru.
compatibility: >
  Requires AWS CLI with devops-guru, CloudWatch, SNS, SSM, CloudFormation,
  and IAM permissions.
---

# Amazon DevOps Guru Diagnostics

## When to use

Any Amazon DevOps Guru investigation — insights, anomaly detection, resource coverage, notifications, cost estimation, or proactive insights.

## Investigation workflow

### Step 1 — Collect and triage

```
aws devops-guru describe-account-health
aws devops-guru describe-account-overview --from-time <epoch>
aws devops-guru list-insights --status-filter ONGOING
aws devops-guru describe-resource-collection-health --resource-collection-type AWS_SERVICE
```

### Step 2 — Domain deep dive

```
aws devops-guru describe-insight --id <insight-id>
aws devops-guru list-anomalies-for-insight --insight-id <insight-id>
aws devops-guru list-recommendations --insight-id <insight-id>
aws devops-guru list-notification-channels
```

### Step 3 — Detailed investigation

```
aws devops-guru describe-organization-health
aws devops-guru get-cost-estimation
aws devops-guru list-events --filters '{"ResourceCollection":{"CloudFormation":{"StackNames":["<stack>"]}}}'
```

Read `references/guardrails.md` before concluding on any DevOps Guru issue.

## Gotchas: Amazon DevOps Guru

- DevOps Guru needs time to learn normal behavior. Anomalies appear after baseline period.
- Resource coverage must be explicitly configured (CloudFormation stacks, tags, or account-wide).
- Insights combine multiple anomalies. A single insight may have many related anomalies.
- Proactive insights predict issues before they occur. They require sufficient historical data.
- Cost estimation is approximate. Actual costs depend on resource count and activity.
- SNS notifications must be configured for alerts. No default notification channel.

## Anti-hallucination rules

1. Always cite insight IDs, anomaly details, or API responses as evidence.
2. DevOps Guru needs learning time. Never expect instant anomaly detection.
3. Resource coverage must be configured. Never assume all resources are monitored.
4. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Insights | A1-A3 | Reactive insights, proactive insights, insight quality |
| B — Anomalies | B1-B2 | Anomaly detection, false positives |
| C — Coverage | C1-C2 | Resource coverage, stack configuration |
| D — Notifications | D1-D2 | SNS channels, notification filtering |
| E — Cost | E1-E2 | Cost estimation, cost optimization |
| F — Proactive | F1-F2 | Proactive insights, recommendations |
| Z — Catch-All | Z1-Z2 | General troubleshooting, IAM issues |
