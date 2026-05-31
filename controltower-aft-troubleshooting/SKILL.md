---
name: controltower-aft-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Control Tower Account
  Factory for Terraform (AFT) problems by analyzing deployment failures,
  pipeline errors, account provisioning, customizations, Terraform state,
  CodePipeline/CodeBuild issues, SSO integration, and following structured runbooks.
  Activate when: AFT deployment failures, pipeline errors, account request issues,
  customization failures, Terraform state problems, CodePipeline/CodeBuild errors,
  SSO integration issues, or the user says something is wrong with AFT.
compatibility: >
  Requires AWS CLI or SDK access with controltower, organizations, codepipeline,
  codebuild, dynamodb, s3, sso-admin, iam, sts, lambda, and cloudformation permissions.
---

# AWS Control Tower AFT Diagnostics

## When to use

Any AWS Control Tower Account Factory for Terraform (AFT) investigation — deployment failures, pipeline errors, account provisioning, account customizations, Terraform state management, provider configuration, CodePipeline/CodeBuild issues, SSO integration, VPC configuration, or upgrade problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws codepipeline list-pipelines --query 'pipelines[?contains(name,`aft`)].{Name:name,Created:created}'
aws codepipeline get-pipeline-state --name aft-account-request --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}'
aws dynamodb scan --table-name aft-request --select COUNT
```

### Step 2 — Domain deep dive

```
aws codebuild list-builds-for-project --project-name aft-account-request --max-items 5
aws codebuild batch-get-builds --ids <build-id> --query 'builds[0].{Status:buildStatus,Phase:currentPhase,Logs:logs.deepLink}'
aws dynamodb get-item --table-name aft-request --key '{"id":{"S":"<account-request-id>"}}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=controltower.amazonaws.com --max-results 20
aws s3 ls s3://aft-backend-<account-id>-<region>/
aws lambda list-functions --query 'Functions[?contains(FunctionName,`aft`)].{Name:FunctionName,Runtime:Runtime,LastModified:LastModified}'
```

Read `references/guardrails.md` before concluding on any AFT issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `codepipeline get-pipeline-state` | Check AFT pipeline execution status |
| `codebuild batch-get-builds` | Get build details and logs |
| `dynamodb get-item` | Check account request status in DynamoDB |
| `s3 ls` | Verify Terraform state backend |
| `controltower list-enabled-controls` | Check Control Tower controls |
| `organizations describe-account` | Verify account provisioning status |
| `lambda get-function` | Check AFT Lambda function configuration |

## Gotchas: AWS Control Tower AFT

- AFT uses a multi-pipeline architecture: account-request, account-provisioning, global-customizations, and account-customizations pipelines. Each can fail independently.
- Terraform state is stored in S3 with DynamoDB locking. State corruption or lock contention causes cascading failures across all AFT operations.
- Account requests are tracked in DynamoDB. The `aft-request` table is the source of truth for account provisioning status — not the pipeline status.
- AFT customizations run in a specific order: global customizations first, then account-specific customizations. Failures in global customizations block account customizations.
- SSO permission sets must exist before AFT can assign them. AFT does not create permission sets — it only assigns existing ones to accounts.
- AFT uses CodePipeline and CodeBuild under the hood. Most "AFT failures" are actually CodePipeline or CodeBuild failures that need to be diagnosed at that layer.
- Upgrading AFT requires careful version compatibility checks. Terraform provider versions, AFT module versions, and Control Tower versions must all be compatible.

## Anti-hallucination rules

1. Always cite specific pipeline names, build IDs, or DynamoDB items as evidence.
2. AFT pipelines and CodePipeline/CodeBuild are different layers. Diagnose at the correct layer.
3. Terraform state issues require careful handling. Never suggest deleting state files.
4. Account provisioning and customization are separate processes. Never conflate them.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Deployment | A1–A2 | AFT deployment failures, pipeline errors |
| B — Account Provisioning | B1–B2 | Account request failures, customization errors |
| C — Terraform | C1–C2 | State issues, provider configuration |
| D — Customizations | D1–D2 | Global customization failures, account customization failures |
| E — CI/CD | E1–E2 | CodePipeline errors, CodeBuild failures |
| F — Integration | F1–F2 | SSO integration, VPC configuration |
| G — Maintenance | G1 | AFT upgrade issues |
| Z — Catch-All | Z1 | General troubleshooting |
