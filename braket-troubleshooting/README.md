# Amazon Braket Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Braket quantum computing problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Braket — quantum task failures, simulator errors, hardware device issues, circuit validation, hybrid jobs, notebook instances, SDK errors, cost management, device availability, and result retrieval.

### Activate When

- Quantum task failures or unexpected results
- Task timeout issues
- Simulator errors
- Hardware QPU device problems
- Circuit validation failures
- Hybrid job failures
- Notebook instance issues
- SDK errors or version incompatibilities
- Cost management concerns
- Device availability questions
- Result retrieval failures
- General Braket errors

---

## Skill Structure

```
braket-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-quantum-task-failures.md
    ├── A2-task-timeout.md
    ├── B1-simulator-errors.md
    ├── B2-hardware-device-issues.md
    ├── C1-circuit-validation.md
    ├── C2-hybrid-job-failures.md
    ├── D1-notebook-instance-issues.md
    ├── D2-sdk-errors.md
    ├── E1-cost-management.md
    ├── E2-device-availability.md
    ├── F1-result-retrieval.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Quantum Tasks** | A1–A2 | Task failures, timeouts |
| **B — Devices** | B1–B2 | Simulator errors, hardware issues |
| **C — Circuits** | C1–C2 | Validation, hybrid jobs |
| **D — Environment** | D1–D2 | Notebook instances, SDK errors |
| **E — Operations** | E1–E2 | Cost management, device availability |
| **F — Results** | F1 | Result retrieval |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `braket:*`, `s3:GetObject`, `s3:ListBucket`, `logs:FilterLogEvents`, `cloudwatch:GetMetricStatistics`, `iam:Get*` permissions

---

## License

MIT-0
