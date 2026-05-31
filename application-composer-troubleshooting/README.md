# AWS Application Composer Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Application Composer problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Application Composer — project sync, template imports, resource configuration, canvas connections, SAM/CloudFormation template generation, local development, IDE extensions, deployments, resource policies, and custom resource cards.

### Activate When

- Project sync failures
- Template import errors
- Resource configuration problems
- Canvas connection issues
- SAM template generation failures
- CloudFormation template errors
- Local development issues
- IDE extension errors
- Deployment failures
- Resource policy issues
- Custom resource card problems
- General Application Composer errors

---

## Skill Structure

```
application-composer-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-project-sync-failures.md
    ├── A2-template-import-errors.md
    ├── B1-resource-configuration-errors.md
    ├── B2-connection-issues.md
    ├── C1-sam-template-generation.md
    ├── C2-cloudformation-template-errors.md
    ├── D1-local-development-issues.md
    ├── D2-ide-extension-errors.md
    ├── E1-deployment-failures.md
    ├── E2-resource-policy-issues.md
    ├── F1-custom-resource-cards.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Project Sync** | A1–A2 | Sync failures, template imports |
| **B — Resources** | B1–B2 | Configuration, connections |
| **C — Templates** | C1–C2 | SAM generation, CloudFormation errors |
| **D — Local Dev** | D1–D2 | Local development, IDE extensions |
| **E — Deployment** | E1–E2 | Deployment failures, resource policies |
| **F — Custom** | F1 | Custom resource cards |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `cloudformation:*`, `sam:*`, `s3:Get*`, `lambda:Get*`, `iam:Get*` permissions
- SAM CLI installed for local development troubleshooting
- Docker installed for local Lambda emulation

---

## License

MIT-0
