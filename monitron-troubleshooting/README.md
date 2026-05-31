# Amazon Monitron Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Monitron problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Monitron — project creation, gateway setup, sensor pairing, data gaps, inference anomalies, false positives, gateway connectivity, WiFi configuration, user management, mobile app issues, and data export.

### Activate When

- Project creation failures
- Gateway setup issues
- Sensor pairing errors
- Sensor data gaps
- Inference anomalies
- False positive alerts
- Gateway connectivity problems
- WiFi configuration issues
- User management errors
- Mobile app issues
- Data export problems
- General Monitron errors

---

## Skill Structure

```
monitron-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-project-creation-failures.md
    ├── A2-gateway-setup-issues.md
    ├── B1-sensor-pairing-errors.md
    ├── B2-sensor-data-gaps.md
    ├── C1-inference-anomalies.md
    ├── C2-false-positive-alerts.md
    ├── D1-gateway-connectivity.md
    ├── D2-wifi-configuration.md
    ├── E1-user-management.md
    ├── E2-mobile-app-issues.md
    ├── F1-data-export.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Project & Gateway** | A1–A2 | Project creation failures, gateway setup issues |
| **B — Sensors** | B1–B2 | Sensor pairing errors, sensor data gaps |
| **C — Inference** | C1–C2 | Inference anomalies, false positive alerts |
| **D — Connectivity** | D1–D2 | Gateway connectivity, WiFi configuration |
| **E — Users** | E1–E2 | User management, mobile app issues |
| **F — Data** | F1 | Data export |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `monitron:*`, `s3:Get*`, `iam:Get*`, `kms:Describe*` permissions
- Physical access to Monitron gateways and sensors for hardware troubleshooting
- Monitron mobile app for device setup and pairing

---

## License

MIT-0
