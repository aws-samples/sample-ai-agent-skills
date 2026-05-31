# AWS Mainframe Modernization Advanced Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Mainframe Modernization advanced problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Mainframe Modernization — Blu Age refactoring, Micro Focus replatforming, application deployment, runtime environments, batch jobs, CICS transactions, data migration, dataset management, performance tuning, monitoring, and hybrid connectivity.

### Activate When

- Blu Age refactoring errors
- Micro Focus replatforming issues
- Application deployment failures
- Runtime environment problems
- Batch job failures
- CICS transaction errors
- Data migration issues
- Dataset management problems
- Performance tuning needs
- Monitoring integration gaps
- Hybrid connectivity issues

---

## Skill Structure

```
mainframe-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-blu-age-refactoring-errors.md
    ├── A2-micro-focus-replatforming.md
    ├── B1-application-deployment-failures.md
    ├── B2-runtime-environment-issues.md
    ├── C1-batch-job-failures.md
    ├── C2-cics-transaction-errors.md
    ├── D1-data-migration-issues.md
    ├── D2-dataset-management.md
    ├── E1-performance-tuning.md
    ├── E2-monitoring-integration.md
    ├── F1-hybrid-connectivity.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Modernization Engines** | A1–A2 | Blu Age refactoring, Micro Focus replatforming |
| **B — Deployment & Runtime** | B1–B2 | Deployment failures, runtime environments |
| **C — Workloads** | C1–C2 | Batch jobs, CICS transactions |
| **D — Data** | D1–D2 | Data migration, dataset management |
| **E — Observability** | E1–E2 | Performance tuning, monitoring |
| **F — Connectivity** | F1 | Hybrid connectivity |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `m2:*`, `s3:*`, `cloudwatch:*`, `logs:*`, `ec2:Describe*`, `iam:Get*` permissions
- Knowledge of mainframe concepts (COBOL, JCL, CICS, VSAM) for effective troubleshooting

---

## License

MIT-0
