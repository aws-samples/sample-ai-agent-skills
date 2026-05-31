# Amazon CodeCatalyst Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CodeCatalyst problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon CodeCatalyst — space creation, project configuration, workflow failures, action errors, dev environments, IDE connectivity, source repositories, pull requests, deployments, environment configuration, AWS connections, IAM roles, and blueprints.

### Activate When

- Space creation failures
- Project configuration issues
- Workflow failures
- Action errors
- Dev environment problems
- IDE connectivity issues
- Source repository errors
- Pull request problems
- Deployment failures
- Environment configuration issues
- AWS connection errors
- IAM role problems
- Blueprint errors
- General CodeCatalyst errors

---

## Skill Structure

```
codecatalyst-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-space-creation.md
    ├── A2-project-configuration.md
    ├── B1-workflow-failures.md
    ├── B2-action-errors.md
    ├── C1-dev-environment-issues.md
    ├── C2-ide-connectivity.md
    ├── D1-source-repository-issues.md
    ├── D2-pull-request-problems.md
    ├── E1-deployment-failures.md
    ├── E2-environment-configuration.md
    ├── F1-connection-to-aws.md
    ├── F2-iam-role-issues.md
    ├── G1-blueprint-errors.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Space & Project** | A1–A2 | Space creation, project configuration |
| **B — Workflows** | B1–B2 | Workflow failures, action errors |
| **C — Dev Environments** | C1–C2 | Dev environment issues, IDE connectivity |
| **D — Source Control** | D1–D2 | Repository issues, pull request problems |
| **E — Deployment** | E1–E2 | Deployment failures, environment configuration |
| **F — AWS Integration** | F1–F2 | AWS connection, IAM roles |
| **G — Blueprints** | G1 | Blueprint errors |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS Builder ID or CodeCatalyst access token
- AWS CLI v2 with `codecatalyst:*`, `iam:Get*`, `cloudtrail:LookupEvents` permissions for connected AWS accounts

---

## License

MIT-0
