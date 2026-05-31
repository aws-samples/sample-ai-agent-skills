---
title: "A2 — AFT Pipeline Errors"
description: "Diagnose and resolve AFT CodePipeline execution failures"
status: active
severity: HIGH
triggers: ["AFT pipeline failed", "pipeline execution error", "pipeline stage failed", "source stage error"]
owner: devops-agent
objective: "Identify and resolve the specific pipeline stage and action causing AFT pipeline failures"
context: "AFT uses multiple CodePipeline pipelines. Each pipeline has source, build, and deploy stages. Pipeline failures can occur at any stage and require stage-level diagnosis."
---

## Phase 1 — Triage

MUST:
- List AFT pipelines and their status: `aws codepipeline list-pipelines --query 'pipelines[?contains(name,`aft`)].name'`
- Get pipeline execution state: `aws codepipeline get-pipeline-state --name <pipeline-name> --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}'`
- Get failed execution details: `aws codepipeline list-pipeline-executions --pipeline-name <name> --max-items 5`

SHOULD:
- Check the specific failed stage and action
- Get CodeBuild build ID from the failed action: `aws codepipeline get-pipeline-execution --pipeline-name <name> --pipeline-execution-id <id>`
- Review CodeBuild logs for the failed build

MAY:
- Check CodeCommit or S3 source for recent changes that triggered the pipeline
- Review pipeline event history in CloudTrail

## Phase 2 — Remediate

MUST:
- Fix the root cause identified in the failed CodeBuild stage
- Verify source repository has valid content (Terraform files, account requests)
- Ensure CodeBuild service role has required permissions

SHOULD:
- Retry the failed pipeline stage after fixing the issue: `aws codepipeline retry-stage-execution --pipeline-name <name> --stage-name <stage> --pipeline-execution-id <id> --retry-mode FAILED_ACTIONS`
- Check for concurrent pipeline executions that may conflict

MAY:
- Enable pipeline execution notifications via SNS
- Review pipeline configuration for stage ordering issues

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "codepipeline list-pipeline-executions: GREEN — read-only execution history"
  - "codepipeline retry-stage-execution: YELLOW — re-executes pipeline stage"
  - "codepipeline start-pipeline-execution: YELLOW — triggers full pipeline execution"
```

## Escalation Conditions

- Pipeline fails at the source stage due to repository access issues
- Multiple pipelines fail simultaneously suggesting infrastructure-level issues
- Pipeline execution hangs indefinitely without progressing or failing
- Pipeline configuration is corrupted and cannot be read

## Data Sensitivity

- `aws codepipeline get-pipeline-state`: LOW — pipeline stage status only
- `aws codepipeline get-pipeline-execution`: MEDIUM — execution details and artifact locations
- CodeBuild logs: HIGH — may contain Terraform output with resource details
- Pipeline configuration: MEDIUM — reveals infrastructure architecture

## Prohibited Actions

- NEVER delete AFT pipelines to fix execution errors
- NEVER manually modify pipeline stage configurations without Terraform
- NEVER retry pipelines without understanding why they failed
- NEVER start multiple pipeline executions simultaneously

## Phase 3 — Rollback

1. If a pipeline retry caused additional issues, wait for it to complete before investigating
2. If pipeline configuration was modified outside Terraform, run `terraform apply` to restore
3. If source repository changes caused the failure, revert the commit
4. Document the pipeline failure and resolution
5. Verify downstream pipelines are not affected

## Common Issues

- symptoms: "Pipeline source stage fails with 'Repository not found'"
  diagnosis: "CodeCommit repository was deleted or renamed, or IAM permissions changed."
  resolution: "Verify repository exists and pipeline service role has codecommit:GetBranch permission."

- symptoms: "Pipeline build stage times out"
  diagnosis: "CodeBuild project cannot reach required resources (Terraform registry, AWS APIs)."
  resolution: "Check VPC configuration if CodeBuild runs in VPC. Verify NAT gateway and internet access."

## Output Format

```yaml
root_cause: "aft_pipeline — <specific_cause>"
evidence:
  - type: pipeline_error
    content: "<pipeline stage, action, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix pipeline stage failure"
  long_term: "Implement pipeline monitoring and alerting"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
