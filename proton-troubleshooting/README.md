# AWS Proton Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Proton problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Proton when the console alone isn't enough — template management, environment provisioning, service deployment, pipeline troubleshooting, component management, template sync, Git integration, and IAM permission issues.

### Activate When

- Environment template registration or publishing failures
- Service template registration or publishing failures
- Template version management issues
- Environment provisioning or update failures
- Service creation or instance deployment failures
- Service pipeline failures
- Component provisioning or update failures
- Component drift detection issues
- Template sync configuration problems
- Git integration or CodeStar connection issues
- IAM role permission errors
- CodeStar connection authorization problems

---

## Skill Structure

```
proton-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-environment-template.md
    ├── A2-service-template.md
    ├── A3-version-issues.md
    ├── B1-provisioning-failures.md
    ├── B2-update-failures.md
    ├── C1-service-creation.md
    ├── C2-pipeline-failures.md
    ├── D1-component-provisioning.md
    ├── D2-drift.md
    ├── E1-template-sync.md
    ├── E2-git-integration.md
    ├── F1-iam-roles.md
    ├── F2-codestar-connections.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Templates** | A1–A3 | Environment template, service template, version issues |
| **B — Environment** | B1–B2 | Provisioning failures, update failures |
| **C — Service** | C1–C2 | Service creation, pipeline failures |
| **D — Components** | D1–D2 | Component provisioning, drift |
| **E — Sync** | E1–E2 | Template sync, Git integration |
| **F — Security** | F1–F2 | IAM roles, CodeStar connections |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering template versioning requirements, provisioning engine differences, self-managed provisioning boundaries, service pipeline optionality, component ownership, template sync prerequisites, IAM role separation, environment output dependencies, template bundle schema, drift detection limitations, CodeStar connection authorization, and deployment ordering.

---

## Investigation Workflow

1. **Triage** — List environments/services, check deployment statuses, review templates
2. **Deep Dive** — Examine template versions, service instances, pipeline outputs
3. **Detailed** — Review CloudTrail events, template sync status, CodeStar connections

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `proton:*`, `iam:GetRole`, `iam:ListAttachedRolePolicies`, `cloudformation:DescribeStacks`, `codepipeline:GetPipelineState`, `codebuild:BatchGetBuilds`, `codestar-connections:ListConnections`, `cloudtrail:LookupEvents`

---

## Usage Examples

```
# List all environments
aws proton list-environments

# Get environment details
aws proton get-environment --name my-env

# List template versions
aws proton list-environment-template-versions --template-name my-template

# Check template sync status
aws proton get-template-sync-status --template-name my-template --template-type ENVIRONMENT

# List CodeStar connections
aws codestar-connections list-connections
```

---

## License

MIT-0
