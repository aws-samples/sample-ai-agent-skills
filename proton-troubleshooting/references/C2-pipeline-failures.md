---
title: "C2 — Proton Service Pipeline Failures"
description: "Diagnose service pipeline creation and execution failures"
status: active
severity: MEDIUM
triggers:
  - "pipeline failed"
  - "pipeline error"
  - "build failed"
  - "deployment pipeline"
  - "CodePipeline error"
owner: devops-agent
objective: "Identify why a Proton service pipeline fails and restore CI/CD functionality"
context: "Proton service pipelines provide CI/CD for service deployments. Pipelines use CodePipeline and CodeBuild. Pipeline configuration is defined in the service template's pipeline/ directory. Pipeline failures affect automated deployments."
---

## Phase 1 — Triage

MUST:
- Check service pipeline status: `aws proton get-service --name <service-name> --query 'service.pipeline'`
- List pipeline outputs: `aws proton list-service-pipeline-outputs --service-name <service-name>`
- Check CodePipeline state: `aws codepipeline get-pipeline-state --name <pipeline-name>`
- Check CodeBuild logs: `aws codebuild batch-get-builds --ids <build-id> --query 'builds[*].{Status:buildStatus,Logs:logs}'`
- Check pipeline IAM role: `aws iam get-role --role-name <pipeline-role-name>`

SHOULD:
- Review CodeBuild build logs for specific errors
- Verify source repository connection is active
- Check if pipeline was recently updated or recreated

MAY:
- Check CodePipeline execution history: `aws codepipeline list-pipeline-executions --pipeline-name <pipeline-name>`
- Verify artifact S3 bucket accessibility

## Phase 2 — Remediate

MUST:
- Fix build errors identified in CodeBuild logs
- Grant missing permissions to the pipeline IAM role
- Ensure source repository connection (CodeStar) is AVAILABLE

SHOULD:
- Verify buildspec.yml in the service repository is correct
- Test builds locally before pushing to the pipeline
- Check pipeline stage transitions for blocked stages

MAY:
- Retry failed pipeline execution: `aws codepipeline retry-stage-execution --pipeline-name <name> --stage-name <stage> --pipeline-execution-id <id> --retry-mode FAILED_ACTIONS`
- Update pipeline configuration through service template update

## Common Issues

- symptoms: "Pipeline source stage fails"
  diagnosis: "CodeStar connection is PENDING or repository access is revoked."
  resolution: "Authorize the CodeStar connection in the AWS Console."

- symptoms: "Pipeline build stage fails"
  diagnosis: "Build errors in CodeBuild — syntax errors, missing dependencies, or test failures."
  resolution: "Check CodeBuild logs for specific errors and fix the build."

- symptoms: "Pipeline deploy stage fails"
  diagnosis: "Deployment permissions or resource creation errors."
  resolution: "Check pipeline role permissions and CloudFormation stack events."

## Output Format

```yaml
root_cause: "pipeline_failure — <specific_cause>"
evidence:
  - type: pipeline_state
    content: "<CodePipeline stage statuses>"
  - type: build_logs
    content: "<CodeBuild error messages>"
  - type: iam_role
    content: "<pipeline role permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix the identified build, permission, or connection issue"
  long_term: "Add build validation and testing stages to the pipeline"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — pipelines may access sensitive deployment configurations
- Service Disruption Risk: HIGH — pipeline failures prevent service deployments
- Compliance Impact: MEDIUM — pipeline failures affect deployment compliance

## Escalation Conditions
- Pipeline fails repeatedly with same error
- Pipeline failure leaves deployment in inconsistent state
- Pipeline permissions prevent deployment completion
- Pipeline timeout causes partial deployment
- Multiple pipelines fail simultaneously

## Data Sensitivity
- Pipeline configurations — INTERNAL
- Deployment parameters — INTERNAL (may contain CONFIDENTIAL data)
- Build artifacts — INTERNAL
- Pipeline execution logs — INTERNAL
- Never expose pipeline credentials or deployment secrets

## Prohibited Actions
- Do NOT retry failed pipelines without understanding the failure
- Do NOT bypass pipeline approval stages
- Do NOT grant pipeline roles overly broad permissions
- Do NOT ignore pipeline failure patterns
- Do NOT modify pipeline configurations during active deployments
- Do NOT store credentials in pipeline configurations

## Phase 3 — Rollback
- Cancel stuck pipeline executions
- Revert deployment to previous version
- Restore pipeline configurations
- Verify service health after pipeline rollback
- Document pipeline failures and root causes
- Update pipeline error handling procedures

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
