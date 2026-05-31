---
title: "H1 — Customizations for Control Tower (CfCT) Pipeline Failures"
description: "Diagnose failures in the Customizations for Control Tower deployment pipeline"
status: active
severity: HIGH
triggers:
  - "CfCT pipeline failed"
  - "Customizations pipeline error"
  - "CodePipeline failure"
  - "Custom StackSet deployment failed"
owner: devops-agent
objective: "Identify and resolve CfCT pipeline failures"
context: "Customizations for Control Tower (CfCT) is an AWS solution that extends CT with custom StackSets and SCPs via a CodePipeline. Failures occur in the pipeline stages: source, build, or deploy. The pipeline is triggered by S3 uploads or CodeCommit pushes."
---

## Phase 1 — Triage

MUST:
- Check CodePipeline execution status: `aws codepipeline get-pipeline-state --name <pipeline-name>`
- Identify the failed stage and action
- Check CodeBuild logs for build failures
- Check CloudFormation StackSet operations for deployment failures

SHOULD:
- Review the manifest.yaml for syntax errors
- Verify the S3 source bucket has the correct configuration package
- Check if custom templates have syntax errors

MAY:
- Check CloudTrail for pipeline-related API calls
- Review the CfCT Lambda function logs

## Phase 2 — Remediate

MUST:
- Fix the identified error (template syntax, manifest, permissions)
- Re-trigger the pipeline after fixes
- Verify the deployment completes successfully

SHOULD:
- Validate CloudFormation templates before uploading
- Test custom SCPs in a sandbox OU first
- Review the CfCT manifest against documentation

MAY:
- Set up pipeline failure notifications via SNS
- Implement template validation in a pre-commit hook

## Common Issues

- symptoms: "Pipeline fails at build stage"
  diagnosis: "Manifest.yaml syntax error or template validation failure."
  resolution: "Fix the manifest or template syntax and re-trigger."

- symptoms: "Pipeline fails at deploy stage"
  diagnosis: "StackSet deployment failed in one or more accounts."
  resolution: "Check failed stack instances for the specific error. Common: permission issues, resource conflicts."

- symptoms: "Pipeline not triggered after S3 upload"
  diagnosis: "S3 event notification or pipeline trigger misconfigured."
  resolution: "Verify the S3 bucket event notification and pipeline source configuration."

## Safety Ratings

```
safety_ratings:
  - "get-pipeline-state: GREEN — read-only pipeline status"
  - "list-stack-set-operations: GREEN — read-only StackSet operation listing"
  - "re-trigger pipeline after fix: YELLOW — recoverable, pipeline can be stopped"
  - "fix manifest.yaml or templates: YELLOW — recoverable, version-controlled changes"
  - "deploy custom SCPs via CfCT: RED — can break governance for all accounts in targeted OUs"
  - "deploy custom StackSets via CfCT: RED — modifies resources across multiple accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Custom SCP deployed via CfCT conflicts with CT-managed SCPs
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- Pipeline deploys to production OUs with no rollback mechanism

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CfCT manifest and templates (expose customization patterns)
- MEDIUM: CodeBuild logs (may contain account IDs and resource details)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If a CfCT deployment breaks workloads, update the manifest to remove the problematic customization and re-trigger the pipeline
- If custom SCPs deployed via CfCT conflict with CT, remove them from the manifest and re-trigger
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Keep previous versions of the CfCT configuration package for rollback
- If StackSet deployment fails in specific accounts, do NOT manually fix — update the template and re-trigger the pipeline

## Output Format

```yaml
root_cause: "cfct_pipeline — <stage> failure: <specific cause>"
evidence:
  - type: codepipeline_status
    content: "<pipeline execution details>"
  - type: codebuild_logs
    content: "<build error details>"
severity: HIGH
mitigation:
  immediate: "Fix the error and re-trigger the pipeline"
  long_term: "Implement pre-deployment validation and monitoring"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
