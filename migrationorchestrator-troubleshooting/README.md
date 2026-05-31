# AWS Migration Hub Orchestrator Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Migration Hub Orchestrator problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Migration Hub Orchestrator — workflow templates, migration workflows, step execution, plugins, SAP migration, and SQL Server migration.

### Activate When

- Workflow template selection issues
- Migration workflow creation failures
- Step execution errors
- Plugin installation or health problems
- SAP migration workflow issues
- SQL Server migration errors
- General Orchestrator errors

---

## Skill Structure

```
migrationorchestrator-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-template-selection.md
    ├── A2-template-configuration.md
    ├── B1-workflow-creation.md
    ├── B2-workflow-execution.md
    ├── C1-step-execution.md
    ├── C2-step-failures.md
    ├── D1-plugin-installation.md
    ├── D2-plugin-health.md
    ├── E1-sap-migration.md
    ├── E2-sap-specific-issues.md
    ├── F1-sqlserver-migration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Templates** | A1–A2 | Selection, configuration |
| **B — Workflows** | B1–B2 | Creation, execution |
| **C — Steps** | C1–C2 | Execution, failures |
| **D — Plugins** | D1–D2 | Installation, health |
| **E — SAP** | E1–E2 | Migration, specific issues |
| **F — SQL Server** | F1 | Migration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `migrationhuborchestrator:*`, `ec2:Describe*`, `ssm:*`, `iam:Get*` permissions

---

## License

MIT-0
