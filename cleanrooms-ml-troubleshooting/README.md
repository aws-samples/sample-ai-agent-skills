# AWS Clean Rooms ML Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Clean Rooms ML problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Clean Rooms ML — training models, configured models, audience generation, lookalike models, collaboration ML configuration, privacy budgets, seed audiences, output configuration, S3 data access, IAM permissions, and differential privacy.

### Activate When

- Training model failures
- Configured model errors
- Audience generation issues
- Lookalike model problems
- Collaboration ML configuration errors
- Privacy budget exhaustion
- Seed audience errors
- Output configuration failures
- S3 data access issues
- IAM permission errors
- Differential privacy problems
- General Clean Rooms ML errors

---

## Skill Structure

```
cleanrooms-ml-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-training-model-failures.md
    ├── A2-configured-model-errors.md
    ├── B1-audience-generation-issues.md
    ├── B2-lookalike-model-errors.md
    ├── C1-collaboration-ml-config.md
    ├── C2-privacy-budget-issues.md
    ├── D1-seed-audience-errors.md
    ├── D2-output-configuration.md
    ├── E1-s3-data-access.md
    ├── E2-iam-permissions.md
    ├── F1-differential-privacy.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Training** | A1–A2 | Training model failures, configured model errors |
| **B — Audience** | B1–B2 | Audience generation issues, lookalike model errors |
| **C — Collaboration** | C1–C2 | Collaboration ML config, privacy budget issues |
| **D — Seed & Output** | D1–D2 | Seed audience errors, output configuration |
| **E — Access** | E1–E2 | S3 data access, IAM permissions |
| **F — Privacy** | F1 | Differential privacy |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `cleanroomsml:*`, `cleanrooms:*`, `s3:Get*`, `iam:Get*`, `kms:Describe*` permissions

---

## License

MIT-0
