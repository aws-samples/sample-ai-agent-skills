# AWS Supply Chain Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Supply Chain problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Supply Chain — instance creation, data integration, demand planning, supply planning, inventory visibility, insights, connectors, and S3 integration.

### Activate When

- Instance creation failures
- Instance configuration issues
- Data integration flow failures
- Data lake dataset issues
- Demand planning errors
- Supply planning problems
- Inventory visibility failures
- Insights generation issues
- Connector failures
- S3 integration problems
- Permission issues
- General Supply Chain errors

---

## Skill Structure

```
supplychain-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-instance-creation.md
    ├── A2-instance-configuration.md
    ├── B1-data-integration-failures.md
    ├── B2-data-lake-issues.md
    ├── C1-demand-planning-errors.md
    ├── C2-supply-planning-issues.md
    ├── D1-inventory-visibility.md
    ├── D2-insights-generation.md
    ├── E1-connector-failures.md
    ├── E2-s3-integration.md
    ├── F1-permission-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1–A2 | Creation, configuration |
| **B — Data** | B1–B2 | Integration failures, data lake |
| **C — Planning** | C1–C2 | Demand planning, supply planning |
| **D — Visibility** | D1–D2 | Inventory visibility, insights |
| **E — Connectors** | E1–E2 | Connector failures, S3 integration |
| **F — Security** | F1 | Permission issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `scn:*`, `s3:GetObject`, `s3:PutObject`, `iam:Get*`, `kms:Describe*` permissions

---

## License

MIT-0
