# Cost Explorer Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Cost Explorer and Budgets problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Cost Explorer and Budgets when the console alone isn't enough — unexpected cost spikes, cost allocation issues, untagged resources, budget alert failures, forecast inaccuracy, budget action failures, anomaly detection, savings recommendations, CUR reports, and organizational billing.

### Activate When

- Unexpected cost increase or spike
- Cost allocation tags not appearing in reports
- Untagged resources causing allocation gaps
- Budget alerts not firing when threshold exceeded
- Budget forecast inaccurate
- Budget actions not executing
- Cost anomaly detection not working
- False positive anomaly alerts
- RI or Savings Plans recommendations seem wrong
- Low RI/SP utilization
- CUR report not appearing in S3
- CUR report delivery failures
- Cannot see linked account costs
- Consolidated billing discrepancies

---

## Skill Structure

```
costexplorer-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-unexpected-costs.md
    ├── A2-cost-allocation.md
    ├── A3-untagged-resources.md
    ├── B1-budget-alerts.md
    ├── B2-forecast-accuracy.md
    ├── B3-action-failures.md
    ├── C1-anomaly-detection.md
    ├── C2-false-positives.md
    ├── D1-ri-sp-recommendations.md
    ├── D2-utilization.md
    ├── E1-cur-setup.md
    ├── E2-report-delivery.md
    ├── F1-linked-account-costs.md
    ├── F2-consolidated-billing.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cost Analysis** | A1–A3 | Unexpected costs, cost allocation, untagged resources |
| **B — Budgets** | B1–B3 | Budget alerts, forecast accuracy, action failures |
| **C — Anomaly** | C1–C2 | Anomaly detection, false positives |
| **D — Savings** | D1–D2 | RI/SP recommendations, utilization |
| **E — Reports** | E1–E2 | CUR setup, report delivery |
| **F — Organization** | F1–F2 | Linked account costs, consolidated billing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
