---
name: codecatalyst-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CodeCatalyst
  problems by analyzing space creation, project configuration,
  workflow failures, action errors, dev environment issues, IDE
  connectivity, source repositories, pull requests, deployments,
  environment configuration, AWS connections, IAM roles, blueprints,
  and following structured runbooks.
  Activate when: space creation failures, project issues, workflow
  errors, action failures, dev environment problems, IDE connectivity
  issues, repository errors, pull request problems, deployment
  failures, environment configuration issues, AWS connection errors,
  IAM role problems, blueprint errors,
  or the user says something is wrong with CodeCatalyst.
compatibility: >
  Requires CodeCatalyst access token or AWS Builder ID, plus
  AWS CLI with codecatalyst, iam, cloudtrail, and cloudwatch
  permissions for connected AWS accounts.
---

# Amazon CodeCatalyst Diagnostics

## When to use

Any Amazon CodeCatalyst investigation — space creation, project configuration, workflow failures, action errors, dev environments, IDE connectivity, source repositories, pull requests, deployments, environment configuration, AWS connections, IAM roles, or blueprint errors.

## Investigation workflow

### Step 1 — Collect and triage

```
aws codecatalyst list-spaces
aws codecatalyst get-space --name <space-name>
aws codecatalyst list-projects --space-name <space-name>
```

### Step 2 — Domain deep dive

```
aws codecatalyst get-project --space-name <space-name> --name <project-name>
aws codecatalyst list-workflow-runs --space-name <space-name> --project-name <project-name> --sort-by START_TIME
aws codecatalyst list-dev-environments --space-name <space-name> --project-name <project-name>
```

### Step 3 — Detailed investigation

```
aws codecatalyst get-workflow-run --space-name <space-name> --project-name <project-name> --id <run-id>
aws codecatalyst get-dev-environment --space-name <space-name> --project-name <project-name> --id <env-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codecatalyst.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any CodeCatalyst issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `codecatalyst list-spaces` | List CodeCatalyst spaces |
| `codecatalyst get-space` | Get space details |
| `codecatalyst list-projects` | List projects in a space |
| `codecatalyst get-project` | Get project details |
| `codecatalyst list-workflow-runs` | List workflow runs |
| `codecatalyst get-workflow-run` | Get workflow run details |
| `codecatalyst list-dev-environments` | List dev environments |
| `codecatalyst get-dev-environment` | Get dev environment details |

## Gotchas: Amazon CodeCatalyst

- CodeCatalyst uses AWS Builder ID for authentication, NOT IAM users. Builder ID is a separate identity system from AWS IAM.
- Spaces are the top-level organizational unit. They are NOT AWS accounts. A space can be connected to multiple AWS accounts.
- Workflows use YAML definitions similar to but NOT identical to GitHub Actions. Do not assume GitHub Actions syntax works directly.
- Dev environments are cloud-based compute instances. They have configurable instance types and storage but are NOT EC2 instances you manage.
- AWS connections link CodeCatalyst spaces to AWS accounts. They require an IAM role in the target account with a trust policy for CodeCatalyst.
- Blueprints are project templates. They generate initial project structure but do NOT continuously manage the project after creation.
- Source repositories in CodeCatalyst are managed Git repositories. They support linking to external repositories (GitHub, Bitbucket) but these are separate from CodeCatalyst-native repos.

## Anti-hallucination rules

1. Always cite specific space names, project names, workflow run IDs, or API responses as evidence.
2. CodeCatalyst uses Builder ID, NOT IAM users. Never suggest IAM user login for CodeCatalyst.
3. Workflows are NOT GitHub Actions. Never assume GitHub Actions syntax or features.
4. Dev environments are NOT EC2 instances. Never suggest EC2 management commands.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Space & Project | A1–A2 | Space creation, project configuration |
| B — Workflows | B1–B2 | Workflow failures, action errors |
| C — Dev Environments | C1–C2 | Dev environment issues, IDE connectivity |
| D — Source Control | D1–D2 | Repository issues, pull request problems |
| E — Deployment | E1–E2 | Deployment failures, environment configuration |
| F — AWS Integration | F1–F2 | Connection to AWS, IAM role issues |
| G — Blueprints | G1 | Blueprint errors |
| Z — Catch-All | Z1 | General troubleshooting |
