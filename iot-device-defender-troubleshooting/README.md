# AWS IoT Device Defender Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT Device Defender problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT Device Defender — audit configuration, audit findings, security profiles, metric anomalies, mitigation actions, custom metrics, alert targets, SNS notifications, ML Detect, dimensions, and fleet indexing integration.

### Activate When

- Audit configuration errors
- Audit finding analysis issues
- Security profile problems
- Metric anomaly detection failures
- Mitigation action errors
- Custom metric issues
- Alert target configuration problems
- SNS notification failures
- ML Detect issues
- Dimension configuration errors
- Fleet indexing integration problems
- General Device Defender errors

---

## Skill Structure

```
iot-device-defender-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-audit-configuration-errors.md
    ├── A2-audit-finding-analysis.md
    ├── B1-detect-security-profile.md
    ├── B2-detect-metric-anomalies.md
    ├── C1-mitigation-action-failures.md
    ├── C2-custom-metric-issues.md
    ├── D1-alert-target-configuration.md
    ├── D2-sns-notification-errors.md
    ├── E1-ml-detect-issues.md
    ├── E2-dimension-configuration.md
    ├── F1-fleet-indexing-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Audit** | A1–A2 | Configuration errors, finding analysis |
| **B — Detect** | B1–B2 | Security profiles, metric anomalies |
| **C — Mitigation** | C1–C2 | Mitigation actions, custom metrics |
| **D — Alerts** | D1–D2 | Alert targets, SNS notifications |
| **E — ML & Dimensions** | E1–E2 | ML Detect, dimensions |
| **F — Integration** | F1 | Fleet indexing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `iot:*`, `sns:Publish`, `iam:PassRole`, `cloudwatch:PutMetricData`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
