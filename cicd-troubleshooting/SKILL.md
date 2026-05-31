---
name: cicd-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS CI/CD pipeline
  problems by analyzing CodePipeline executions, CodeBuild projects,
  CodeDeploy deployments, artifact management, and following
  structured runbooks. Activate when: pipeline execution failures,
  source stage errors, approval timeouts, cross-account pipeline
  issues, build failures, build timeouts, out-of-memory builds,
  Docker build issues, deployment failures, rollback problems,
  health check failures, lifecycle hook errors, artifact upload
  or download failures, encryption issues, S3 permissions,
  GitHub/CodeCommit source problems, ECR image issues, Lambda
  deploy failures, pipeline or deployment notifications not
  working, slow builds, slow deployments, or the user says
  something is wrong with their CI/CD pipeline without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI with codepipeline, codebuild, codedeploy, s3,
  kms, iam, events, sns, ecr, and cloudtrail permissions.
  Optional: GitHub CLI, Docker CLI.
---

# CI/CD Pipeline Diagnostics

## When to use

Any AWS CI/CD investigation where the console alone is insufficient — CodePipeline execution failures, CodeBuild build errors, CodeDeploy deployment problems, artifact management, cross-account pipelines, notification issues, or performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
# Pipeline state — current status of all stages and actions
aws codepipeline get-pipeline-state --name <pipeline-name>

# Pipeline execution history — recent executions and their outcomes
aws codepipeline list-pipeline-executions --pipeline-name <pipeline-name> --max-items 10

# Specific execution details
aws codepipeline get-pipeline-execution --pipeline-name <pipeline-name> --pipeline-execution-id <execution-id>

# CodeBuild build details — phases, logs, environment
aws codebuild batch-get-builds --ids <build-id>

# CodeDeploy deployment details — status, instances, errors
aws codedeploy get-deployment --deployment-id <deployment-id>

# CodeDeploy deployment targets
aws codedeploy list-deployment-targets --deployment-id <deployment-id>
```

Triage returns:
- Pipeline execution status across all stages (Source, Build, Deploy, Approval)
- Failed action name, error code, and error message
- Build phase details (SUBMITTED, QUEUED, PROVISIONING, DOWNLOAD_SOURCE, INSTALL, PRE_BUILD, BUILD, POST_BUILD, UPLOAD_ARTIFACTS, FINALIZING)
- Deployment status, deployment type (in-place vs blue/green), and instance health
- Artifact locations and encryption configuration

If the pipeline itself is misconfigured (no stages, bad IAM role), that IS the root cause domain. Don't chase downstream build/deploy symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# CodeBuild logs from CloudWatch
aws logs get-log-events --log-group-name /aws/codebuild/<project-name> --log-stream-name <build-id>

# CodeDeploy instance-level deployment status
aws codedeploy get-deployment-target --deployment-id <deployment-id> --target-id <instance-id>

# Pipeline artifact store configuration
aws codepipeline get-pipeline --name <pipeline-name> --query 'pipeline.artifactStore'

# S3 artifact bucket permissions
aws s3api get-bucket-policy --bucket <artifact-bucket>

# KMS key policy for artifact encryption
aws kms describe-key --key-id <key-id>
aws kms get-key-policy --key-id <key-id> --policy-name default

# CloudTrail for pipeline API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codepipeline.amazonaws.com --max-results 20

# EventBridge rules for pipeline triggers
aws events list-rules --name-prefix "codepipeline"
```

Read `references/cicd-guardrails.md` before concluding on any CI/CD issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# Pipeline definition — full structure including all stages, actions, and configuration
aws codepipeline get-pipeline --name <pipeline-name>

# CodeBuild project configuration — environment, source, artifacts, cache
aws codebuild batch-get-projects --names <project-name>

# CodeDeploy application and deployment group configuration
aws codedeploy get-deployment-group --application-name <app-name> --deployment-group-name <dg-name>

# IAM role policies for pipeline service role
aws iam get-role --role-name <pipeline-role>
aws iam list-attached-role-policies --role-name <pipeline-role>
aws iam list-role-policies --role-name <pipeline-role>

# CodeBuild service role policies
aws iam list-attached-role-policies --role-name <codebuild-role>

# Cross-account trust relationships
aws iam get-role --role-name <cross-account-role> --query 'Role.AssumeRolePolicyDocument'

# SNS topic subscriptions for notifications
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>

# ECR image details
aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws codepipeline get-pipeline-state` | Current status of all stages and actions |
| `aws codepipeline get-pipeline-execution` | Details of a specific execution |
| `aws codepipeline list-pipeline-executions` | Recent execution history |
| `aws codepipeline get-pipeline` | Full pipeline definition and configuration |
| `aws codebuild batch-get-builds` | Build details, phases, logs, environment |
| `aws codebuild batch-get-projects` | Project configuration, source, cache, environment |
| `aws codedeploy get-deployment` | Deployment status, type, configuration |
| `aws codedeploy get-deployment-target` | Instance-level deployment status and lifecycle events |
| `aws codedeploy list-deployment-targets` | All targets in a deployment |
| `aws codedeploy get-deployment-group` | Deployment group config, auto-rollback, alarms |
| `aws s3api get-bucket-policy` | Artifact bucket permissions |
| `aws kms describe-key` | Artifact encryption key details |
| `aws events list-rules` | EventBridge rules for pipeline triggers |
| `aws logs get-log-events` | CodeBuild logs from CloudWatch |
| CloudTrail | Pipeline, build, and deployment API events |

## Gotchas: CI/CD Pipeline

These are the mistakes commonly made during CI/CD troubleshooting.

- Pipeline execution order matters: if multiple commits are pushed rapidly, CodePipeline processes them sequentially. A new execution supersedes an in-progress one in the same stage, causing the older execution to be SUPERSEDED (not failed). This is expected behavior, not an error.
- CodeBuild timeout defaults to 60 minutes. Maximum is 8 hours (480 minutes). If your build consistently times out, check the `timeoutInMinutes` setting in the project configuration. Long-running builds (Docker layer caching, large test suites) often need this increased.
- CodeDeploy has two deployment types: in-place and blue/green. In-place updates instances one at a time (or in batches). Blue/green creates a new set of instances, shifts traffic, then terminates the old set. The troubleshooting path is completely different for each type.
- Artifact encryption: CodePipeline encrypts artifacts in S3 using AWS-managed keys (aws/s3) by default, or a customer-managed KMS key. Cross-account pipelines MUST use a customer-managed KMS key — the default aws/s3 key cannot be shared across accounts.
- Cross-account pipelines require KMS key sharing. The KMS key policy must grant `kms:Decrypt` and `kms:DescribeKey` to the target account's role. The artifact bucket policy must also allow cross-account access. Missing either one causes "Access Denied" on artifact download.
- CodeBuild local caching can dramatically speed up builds. Three modes: `SOURCE_CACHE` (caches source), `DOCKER_LAYER_CACHE` (caches Docker layers), `CUSTOM_CACHE` (caches specified paths). Docker layer cache requires privileged mode enabled. Cache is per-build host and not guaranteed between builds.
- buildspec.yml phases execute in order: install → pre_build → build → post_build. If a phase fails, subsequent phases are skipped EXCEPT post_build, which always runs (with `CODEBUILD_BUILD_SUCCEEDING=0`). Use post_build for cleanup that must happen regardless of build success.
- appspec.yml hooks execute in a specific order for EC2: BeforeInstall → AfterInstall → ApplicationStart → ValidateService. For Lambda: BeforeAllowTraffic → AfterAllowTraffic. For ECS: BeforeInstall → AfterInstall → AfterAllowTestTraffic → BeforeAllowTraffic → AfterAllowTraffic. Hook scripts that exit non-zero fail the deployment.
- CodeDeploy rollback triggers: automatic rollback can be configured on deployment failure OR CloudWatch alarm threshold breach. If an alarm is in INSUFFICIENT_DATA state, it does NOT trigger a rollback — only ALARM state does. Misconfigured alarms cause rollbacks to not fire when expected.
- Pipeline source detection: polling checks for changes every minute (less efficient, adds latency). Webhooks provide near-instant detection. EventBridge (CloudWatch Events) is the recommended approach for CodeCommit and S3 sources. GitHub connections use webhooks via AWS CodeStar Connections.
- CodeBuild environment variables: `CODEBUILD_BUILD_SUCCEEDING` is "1" during build, "0" if any phase failed. `CODEBUILD_SRC_DIR` is the source directory. `CODEBUILD_BUILD_ID` contains the build ID. These are available in all phases and useful for conditional logic.
- CodeDeploy agent must be running on EC2 instances for EC2/On-Premises deployment type. The agent polls CodeDeploy for deployment instructions. If the agent is stopped or not installed, the deployment hangs at the instance level with no error — it just waits indefinitely.
- Pipeline artifacts are stored in S3 with a specific key structure. Each action's output artifact is a ZIP file. Large artifacts (>500MB) can cause timeout issues during upload/download. Consider using S3 references instead of passing large artifacts between stages.
- CodeBuild VPC configuration: if the build project is in a VPC, it needs a NAT gateway or VPC endpoints to reach AWS services (S3, ECR, CloudWatch Logs). Without internet access, builds fail at DOWNLOAD_SOURCE or when pulling Docker images.
- CodeDeploy minimum healthy hosts: the deployment configuration controls how many instances must remain healthy during deployment. If `minimumHealthyHosts` is set too high and instances fail health checks, the deployment gets stuck because it can't take enough instances out of service.

## Anti-hallucination rules

1. Always cite specific AWS CLI output, build logs, deployment events, or CloudTrail entries as evidence.
2. Never claim CodePipeline can run stages in parallel by default — stages execute sequentially. Only actions within a stage can run in parallel.
3. Never suggest modifying CodeDeploy agent configuration files directly on managed instances — use Systems Manager or user data scripts.
4. Never claim CodeBuild caches are persistent across all builds — local caches are per-build host and not guaranteed.
5. Never assume cross-account pipelines work with the default AWS-managed S3 encryption key — they require a customer-managed KMS key.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Pipeline | A1–A4 | Execution failures, source stage errors, approval timeout, cross-account pipeline |
| B — CodeBuild | B1–B4 | Build failures, timeout, OOM, Docker build issues |
| C — CodeDeploy | C1–C4 | Deployment failures, rollback, health check failures, lifecycle hook errors |
| D — Artifacts | D1–D3 | Artifact upload/download, encryption, S3 permissions |
| E — Integration | E1–E3 | GitHub/CodeCommit source, ECR image, Lambda deploy |
| F — Notifications | F1–F2 | Pipeline notifications, deployment notifications |
| G — Performance | G1–G2 | Slow builds, slow deployments |
| Z — Catch-All | Z1 | General CI/CD troubleshooting |
