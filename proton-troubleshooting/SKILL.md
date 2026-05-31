---
name: proton-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Proton problems by analyzing
  environment templates, service templates, version management, environment
  provisioning, service creation, pipeline failures, component provisioning,
  drift detection, template sync, Git integration, IAM roles, and CodeStar
  connections using structured runbooks. Activate when: environment or service
  template issues, template version failures, environment provisioning or update
  failures, service creation or pipeline failures, component provisioning or
  drift issues, template sync or Git integration problems, IAM role permission
  errors, CodeStar connection issues, or the user says something is wrong with
  AWS Proton without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Proton, IAM, CloudFormation, CodePipeline,
  CodeBuild, CodeStar Connections, S3, CloudTrail, and optionally Terraform
  permissions. CodeStar connection for Git repository access.
---

# AWS Proton Diagnostics

## When to use

Any AWS Proton investigation where the console alone is insufficient — template management, environment provisioning, service deployment, pipeline troubleshooting, component management, template sync, Git integration, or IAM permission issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws proton list-environments --query 'environments[*].{Name:name,Status:deploymentStatus,Template:templateName}'
version: "1.0.0"
last_updated: "2025-04-12"
aws proton list-services --query 'services[*].{Name:name,Status:status,Template:templateName}'
version: "1.0.0"
last_updated: "2025-04-12"
aws proton list-environment-templates
aws proton list-service-templates
aws proton get-environment --name <env-name>
aws proton get-service --name <service-name>
```

### Step 2 — Domain deep dive

```
aws proton get-environment --name <env-name> --query '{Status:deploymentStatus,StatusMessage:deploymentStatusMessage}'
aws proton list-environment-template-versions --template-name <template-name>
aws proton list-service-template-versions --template-name <template-name>
aws proton get-service-instance --name <instance-name> --service-name <service-name>
aws proton list-service-pipeline-outputs --service-name <service-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=proton.amazonaws.com --max-results 20
aws proton list-environment-outputs --environment-name <env-name>
aws proton get-template-sync-status --template-name <template-name> --template-type ENVIRONMENT
aws codestar-connections list-connections
aws codepipeline get-pipeline-state --name <pipeline-name>
```

Read `references/guardrails.md` before concluding on any AWS Proton issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `proton list-environments` | List all environments and status |
| `proton get-environment` | Full environment details and deployment status |
| `proton list-services` | List all services and status |
| `proton get-service` | Full service details |
| `proton get-service-instance` | Service instance details |
| `proton list-environment-templates` | List environment templates |
| `proton list-service-templates` | List service templates |
| `proton get-template-sync-status` | Template sync configuration status |
| `proton list-environment-outputs` | Environment CloudFormation outputs |
| `codestar-connections list-connections` | Git repository connections |
| `codepipeline get-pipeline-state` | Service pipeline status |
| `cloudtrail lookup-events` | Recent Proton API events |

## Gotchas: AWS Proton

- Proton uses templates to define infrastructure — environment templates define shared resources, service templates define per-service resources. Templates are versioned and must be published before use.
- Template versions must be in PUBLISHED status to be used — DRAFT versions cannot be deployed. Registration and publishing are separate steps.
- Environment provisioning uses CloudFormation or Terraform — the provisioning engine is determined by the template. CloudFormation is AWS-managed; Terraform requires self-managed provisioning.
- Service pipelines are optional — not all service templates include pipelines. Pipeline configuration is part of the service template.
- Components are customer-managed infrastructure attached to services — they allow extending service instances without modifying templates.
- Template sync connects templates to Git repositories — changes pushed to the repo automatically create new template versions. Requires CodeStar connection.
- Proton IAM roles: service role (for Proton operations), pipeline role (for CI/CD), and component role (for component provisioning) are separate.
- Environment outputs are passed to service templates as inputs — if environment outputs change, dependent services may need redeployment.
- Self-managed provisioning (Terraform) requires external provisioning infrastructure — Proton sends provisioning requests but doesn't execute Terraform directly.
- Template bundles must follow the Proton schema format — incorrect schema, missing manifests, or invalid Jinja/HCL templates cause registration failures.
- Drift detection identifies differences between desired and actual state — but does not auto-remediate. Manual intervention is required.
- CodeStar connections must be in AVAILABLE status — PENDING connections require manual authorization in the console.

## Anti-hallucination rules

1. Always cite specific template versions, deployment statuses, or API responses as evidence. Never guess at Proton configurations.
2. Proton does NOT execute Terraform directly for self-managed provisioning. Never suggest Proton runs terraform apply.
3. Template versions must be PUBLISHED before deployment. Never suggest deploying DRAFT versions.
4. CodeStar connections require manual authorization. Never suggest they auto-activate.
5. Components are customer-managed, not Proton-managed. Never suggest Proton auto-provisions components.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Templates | A1-A3 | Environment template, service template, version issues |
| B — Environment | B1-B2 | Provisioning failures, update failures |
| C — Service | C1-C2 | Service creation, pipeline failures |
| D — Components | D1-D2 | Component provisioning, drift |
| E — Sync | E1-E2 | Template sync, Git integration |
| F — Security | F1-F2 | IAM roles, CodeStar connections |
| Z — Catch-All | Z1 | General troubleshooting |
