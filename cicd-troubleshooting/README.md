# CI/CD Pipeline Diagnostics Skill

Agent skill for investigating and troubleshooting AWS CI/CD pipeline problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS CI/CD services when the AWS Console alone isn't enough — CodePipeline execution failures, CodeBuild build errors, CodeDeploy deployment problems, artifact management, cross-account pipelines, source integrations (GitHub, CodeCommit, ECR), notification configuration, and performance optimization.

### Activate When

- Pipeline execution fails or gets stuck
- Source stage cannot detect or pull changes
- Approval stage times out or is misconfigured
- Cross-account pipeline has access denied errors
- CodeBuild build fails during any phase
- CodeBuild times out or runs out of memory
- Docker builds fail in CodeBuild
- CodeDeploy deployment fails or rolls back unexpectedly
- Health checks fail during deployment
- Lifecycle hook scripts error out
- Artifacts fail to upload or download between stages
- Artifact encryption or KMS key issues
- S3 artifact bucket permission errors
- GitHub or CodeCommit source integration breaks
- ECR image pull fails during build or deploy
- Lambda deployment through CodeDeploy fails
- Pipeline or deployment notifications not firing
- Builds are slower than expected
- Deployments take too long
- The user says something is wrong with their CI/CD pipeline without naming specific symptoms

---

## Skill Structure

```
cicd-troubleshooting/
├── SKILL.md                                # Main skill definition and investigation workflow
├── README.md                               # This file
└── references/
    ├── cicd-guardrails.md                  # 12 anti-misdiagnosis guardrails
    ├── cicd-hallucination-patterns.yaml    # 8 common LLM hallucination patterns
    ├── A1-pipeline-execution-failures.md
    ├── A2-source-stage-errors.md
    ├── A3-approval-timeout.md
    ├── A4-cross-account-pipeline.md
    ├── B1-build-failures.md
    ├── B2-build-timeout.md
    ├── B3-build-oom.md
    ├── B4-docker-build-issues.md
    ├── C1-deployment-failures.md
    ├── C2-deployment-rollback.md
    ├── C3-health-check-failures.md
    ├── C4-lifecycle-hook-errors.md
    ├── D1-artifact-upload-download.md
    ├── D2-artifact-encryption.md
    ├── D3-s3-artifact-permissions.md
    ├── E1-github-codecommit-source.md
    ├── E2-ecr-image-issues.md
    ├── E3-lambda-deploy.md
    ├── F1-pipeline-notifications.md
    ├── F2-deployment-notifications.md
    ├── G1-slow-builds.md
    ├── G2-slow-deployments.md
    └── Z1-general-troubleshooting.md
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Pipeline** | A1–A4 | Execution failures, source stage errors, approval timeout, cross-account pipeline |
| **B — CodeBuild** | B1–B4 | Build failures, timeout, OOM, Docker build issues |
| **C — CodeDeploy** | C1–C4 | Deployment failures, rollback, health check failures, lifecycle hook errors |
| **D — Artifacts** | D1–D3 | Artifact upload/download, encryption, S3 permissions |
| **E — Integration** | E1–E3 | GitHub/CodeCommit source, ECR image, Lambda deploy |
| **F — Notifications** | F1–F2 | Pipeline notifications, deployment notifications |
| **G — Performance** | G1–G2 | Slow builds, slow deployments |
| **Z — Catch-All** | Z1 | General CI/CD troubleshooting |

---

## Guardrails & Anti-Hallucination

The skill includes two guardrail files that prevent common misdiagnosis:

### CI/CD Guardrails (`cicd-guardrails.md`)
12 rules covering: pipeline execution superseding behavior, CodeBuild timeout limits, CodeDeploy deployment types, artifact encryption requirements, cross-account KMS key sharing, CodeBuild local caching modes, buildspec.yml phase execution order, appspec.yml hook execution order, CodeDeploy rollback trigger behavior, source detection methods, CodeBuild VPC networking requirements, and CodeDeploy minimum healthy hosts configuration.

### Hallucination Patterns (`cicd-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about AWS CI/CD, including:
- Claiming CodePipeline stages can run in parallel (stages are sequential)
- Suggesting cross-account pipelines work with default S3 encryption (requires customer-managed KMS)
- Claiming CodeBuild local cache is persistent across all builds (per-host, not guaranteed)
- Confusing in-place and blue/green deployment rollback behavior
- Assuming CodeDeploy agent is optional for EC2 deployments (required)
- Claiming buildspec.yml post_build phase is skipped on failure (it always runs)
- Suggesting CodeBuild in VPC can access internet without NAT (requires NAT or VPC endpoints)
- Assuming pipeline source polling and webhooks are equivalent (different latency and cost)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using AWS CLI (`codepipeline get-pipeline-state`, `codebuild batch-get-builds`, `codedeploy get-deployment`). Classify the failure domain.

### Phase 2 — Remediate
Deep dive into the specific domain using targeted commands, CloudWatch Logs, CloudTrail events, or S3/KMS configuration. Apply the fix.

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI with CodePipeline, CodeBuild, CodeDeploy, S3, KMS, IAM, EventBridge, SNS, ECR, and CloudTrail permissions
- Access to CloudWatch Logs for build logs
- For GitHub sources: AWS CodeStar Connections or OAuth token configured
- For ECR sources: ECR repository permissions
- For cross-account: IAM roles with appropriate trust policies in both accounts

---

## Usage Examples

### Pipeline Execution Failure
```
My pipeline "prod-deploy" in us-east-1 failed at the Deploy stage.
Check the pipeline state, deployment status, and instance health.
```

### CodeBuild Timeout
```
My CodeBuild project "api-build" keeps timing out after 60 minutes.
The build runs Docker commands and a large test suite. Check the
build phases and timeout configuration.
```

### Cross-Account Pipeline
```
My cross-account pipeline gets "Access Denied" when the Deploy stage
tries to deploy to the production account. Check KMS key policies,
artifact bucket permissions, and cross-account IAM roles.
```

### Deployment Rollback
```
CodeDeploy keeps rolling back my deployment to the "web-app" deployment
group. The rollback happens after the ApplicationStart hook. Check
lifecycle events, health checks, and rollback configuration.
```

### General Triage
```
Something is wrong with my CI/CD pipeline but I'm not sure what.
Builds sometimes fail, deployments are slow. Run a general
investigation and follow whichever runbook matches the symptoms.
```

---

## License

MIT-0
