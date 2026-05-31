# AWS Audit Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Audit Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Audit Manager — assessment creation, frameworks, controls, evidence collection, delegation, reports, and custom frameworks.

### Activate When

- Assessment creation failures
- Framework selection or configuration issues
- Control evaluation not working
- Evidence not being collected
- Delegation setup failures
- Report generation errors
- Custom framework issues
- General Audit Manager errors

---

## Skill Structure

```
auditmanager-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-assessment-creation.md
    ├── A2-assessment-configuration.md
    ├── B1-standard-frameworks.md
    ├── B2-custom-frameworks.md
    ├── C1-control-evaluation.md
    ├── C2-control-status.md
    ├── D1-automated-evidence.md
    ├── D2-manual-evidence.md
    ├── D3-evidence-quality.md
    ├── E1-delegation-setup.md
    ├── E2-delegate-access.md
    ├── F1-report-generation.md
    ├── F2-report-access.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Assessment** | A1–A2 | Creation, configuration |
| **B — Frameworks** | B1–B2 | Standard, custom |
| **C — Controls** | C1–C2 | Evaluation, status |
| **D — Evidence** | D1–D3 | Automated, manual, quality |
| **E — Delegation** | E1–E2 | Setup, access |
| **F — Reports** | F1–F2 | Generation, access |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `auditmanager:*`, `config:Describe*`, `s3:*`, `iam:Get*` permissions
- AWS Config enabled for automated evidence collection

---

## License

MIT-0
