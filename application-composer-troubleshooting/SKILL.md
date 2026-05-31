---
name: application-composer-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Application Composer
  problems by analyzing project sync, template imports, resource configuration,
  connections, SAM/CloudFormation template generation, local development,
  IDE extensions, deployments, resource policies, and custom resource cards.
  Activate when: project sync failures, template import errors, resource
  configuration problems, connection issues, template generation failures,
  local development errors, IDE extension problems, deployment failures,
  or the user says something is wrong with Application Composer.
compatibility: >
  Requires AWS CLI or SDK access with cloudformation, sam, s3,
  lambda, apigateway, iam, and cloudwatch permissions.
---

# AWS Application Composer Diagnostics

## When to use

Any AWS Application Composer investigation — project sync, template imports, resource configuration, canvas connections, SAM/CloudFormation template generation, local development, IDE extension issues, deployments, resource policies, or custom resource cards.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudformation validate-template --template-body file://template.yaml
aws cloudformation list-stacks --stack-status-filter CREATE_FAILED UPDATE_FAILED ROLLBACK_COMPLETE
sam validate --template-file template.yaml --lint
```

### Step 2 — Domain deep dive

```
aws cloudformation describe-stack-events --stack-name <stack-name> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
aws cloudformation get-template --stack-name <stack-name> --template-stage Processed
sam build --debug 2>&1 | head -50
```

### Step 3 — Detailed investigation

```
aws cloudformation describe-stack-resources --stack-name <stack-name>
aws cloudformation detect-stack-drift --stack-name <stack-name>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudformation.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Application Composer issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `cloudformation validate-template` | Validate template syntax |
| `cloudformation describe-stack-events` | Check deployment event history |
| `cloudformation get-template` | Retrieve processed template |
| `sam validate` | Validate SAM template with linting |
| `sam build` | Build SAM application locally |
| `sam local invoke` | Test Lambda functions locally |
| `sam deploy` | Deploy SAM application |

## Gotchas: AWS Application Composer

- Application Composer generates SAM templates by default. CloudFormation templates require explicit export. SAM is a superset of CloudFormation with Transform: AWS::Serverless-2016-10-31.
- Project sync writes changes to local files in real-time. If the local project has unsaved changes or git conflicts, sync can overwrite or fail silently.
- Canvas connections create IAM policies and environment variables automatically. Removing a connection does NOT remove the generated IAM statements — manual cleanup is required.
- Template imports must be valid YAML/JSON. Application Composer silently drops unsupported resource types or properties it cannot render on the canvas.
- Local development with SAM CLI requires Docker for Lambda emulation. Missing Docker or incorrect Docker socket configuration causes cryptic build failures.
- IDE extension (VS Code Toolkit) requires AWS credentials configured. Extension errors often stem from expired or missing credentials, not Application Composer bugs.
- Custom resource cards only support resources with a defined CloudFormation schema. Custom::* resources render as generic cards with limited property editing.

## Anti-hallucination rules

1. Always cite specific template paths, stack names, or error messages as evidence.
2. SAM templates and CloudFormation templates have different syntax. Never conflate Transform sections.
3. Application Composer canvas state and the generated template can diverge. Always verify the template file.
4. Connection-generated IAM policies are additive. Never claim removing a connection cleans up policies.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Project Sync | A1–A2 | Sync failures, template import errors |
| B — Resources | B1–B2 | Configuration errors, connection issues |
| C — Templates | C1–C2 | SAM generation, CloudFormation errors |
| D — Local Dev | D1–D2 | Local development, IDE extension errors |
| E — Deployment | E1–E2 | Deployment failures, resource policy issues |
| F — Custom | F1 | Custom resource cards |
| Z — Catch-All | Z1 | General troubleshooting |
