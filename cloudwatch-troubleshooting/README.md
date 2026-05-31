# CloudWatch Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CloudWatch problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudWatch when the console alone isn't enough — missing logs, log group retention costs, subscription filter failures, metric gaps, custom metric publishing errors, metric math issues, alarm misfires, INSUFFICIENT_DATA states, Synthetics canary failures, Log Insights query problems, Contributor Insights cost surprises, cross-account observability setup, and CloudWatch Agent configuration.

### Activate When

- Logs not appearing in log group
- Log group retention not set (infinite storage costs)
- Subscription filter not delivering to destination
- Cross-account log delivery failures
- Metrics missing or delayed
- Custom metric publishing errors (PutMetricData)
- Metric math expression returning unexpected results
- Alarm stuck in INSUFFICIENT_DATA
- Alarm not triggering despite threshold breach
- Composite alarm not evaluating correctly
- Alarm actions (SNS, Auto Scaling, EC2) not firing
- Synthetics canary failing
- Canary cannot reach VPC endpoints
- Canary script runtime errors
- Log Insights query syntax errors
- Log Insights query timing out
- Contributor Insights rule not matching events
- Contributor Insights unexpected costs
- Cross-account observability not working
- CloudWatch Agent not publishing metrics or logs

---

## Skill Structure

```
cloudwatch-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-missing-logs.md
    ├── A2-log-group-retention.md
    ├── A3-subscription-filter-issues.md
    ├── A4-cross-account-log-delivery.md
    ├── B1-missing-metrics.md
    ├── B2-custom-metric-publishing.md
    ├── B3-metric-math-errors.md
    ├── C1-insufficient-data.md
    ├── C2-alarm-not-triggering.md
    ├── C3-composite-alarms.md
    ├── C4-alarm-actions.md
    ├── D1-canary-failures.md
    ├── D2-canary-vpc-issues.md
    ├── D3-canary-script-errors.md
    ├── E1-query-errors.md
    ├── E2-query-performance.md
    ├── F1-rule-configuration.md
    ├── F2-high-cost.md
    ├── G1-observability-access-manager.md
    ├── G2-cross-account-metrics-logs.md
    ├── H1-agent-issues.md
    ├── H2-unified-agent-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── cloudwatch-guardrails.md
    └── cloudwatch-hallucination-patterns.yaml
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Logs** | A1–A4 | Missing logs, log group retention, subscription filters, cross-account log delivery |
| **B — Metrics** | B1–B3 | Missing metrics, custom metric publishing, metric math errors |
| **C — Alarms** | C1–C4 | INSUFFICIENT_DATA, alarm not triggering, composite alarms, alarm actions |
| **D — Synthetics** | D1–D3 | Canary failures, canary VPC issues, canary script errors |
| **E — Log Insights** | E1–E2 | Query errors, query performance |
| **F — Contributor Insights** | F1–F2 | Rule configuration, high cost |
| **G — Cross-Account** | G1–G2 | Observability access manager, cross-account metrics/logs |
| **H — Agent** | H1–H2 | CloudWatch Agent issues, unified agent configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
