# AWS Fault Injection Service Diagnostics Skill

Agent skill for investigating and troubleshooting AWS FIS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS FIS — experiment templates, target selection, action execution, stop conditions, IAM roles, and experiment execution.

### Activate When

- Experiment template creation errors
- Target resolution failures (no targets found)
- Action execution failures
- Stop conditions not triggering or false triggers
- IAM permission errors during experiments
- Experiment stuck or failed state
- General FIS errors

---

## Skill Structure

```
fis-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-template-creation.md
    ├── A2-template-validation.md
    ├── B1-target-selection.md
    ├── B2-target-resolution.md
    ├── C1-action-configuration.md
    ├── C2-action-execution.md
    ├── C3-action-dependencies.md
    ├── D1-stop-condition-setup.md
    ├── D2-stop-condition-triggers.md
    ├── E1-role-configuration.md
    ├── E2-permission-issues.md
    ├── F1-experiment-failures.md
    ├── F2-experiment-monitoring.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Templates** | A1–A2 | Creation, validation |
| **B — Targets** | B1–B2 | Selection, resolution |
| **C — Actions** | C1–C3 | Configuration, execution, dependencies |
| **D — Stop Conditions** | D1–D2 | Setup, triggers |
| **E — IAM** | E1–E2 | Role config, permissions |
| **F — Execution** | F1–F2 | Failures, monitoring |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `fis:*`, `ec2:*`, `ecs:*`, `rds:*`, `iam:Get*`, `cloudwatch:Describe*` permissions

---

## License

MIT-0
