---
title: "A1 — Pipeline Execution Failures"
description: "Diagnose and resolve CodePipeline execution failures"
status: active
severity: HIGH
triggers:
  - "pipeline failed"
  - "execution failed"
  - "stage failed"
  - "action failed"
  - "InternalError"
owner: devops-agent
objective: "Identify why the pipeline execution failed and resolve the issue"
context: "CodePipeline executions fail when a stage or action encounters an error. Failures can stem from IAM permissions, misconfigured actions, downstream service errors (CodeBuild, CodeDeploy), or infrastructure issues. The pipeline state and execution history provide the starting point for diagnosis."
---

## Phase 1 — Triage

MUST:
- Check pipeline state for failed stages/actions: `aws codepipeline get-pipeline-state --name <pipeline-name>`
- Get recent execution history: `aws codepipeline list-pipeline-executions --pipeline-name <pipeline-name> --max-items 10`
- Get failed execution details: `aws codepipeline get-pipeline-execution --pipeline-name <pipeline-name> --pipeline-execution-id <execution-id>`
- Check the pipeline service role permissions: `aws iam get-role --role-name <pipeline-role>`

SHOULD:
- Check CloudTrail for pipeline API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codepipeline.amazonaws.com --max-results 20`
- Verify the pipeline definition is valid: `aws codepipeline get-pipeline --name <pipeline-name>`
- Check if the execution was SUPERSEDED (not actually failed): look for SUPERSEDED status in execution list

MAY:
- Check AWS Health Dashboard for CodePipeline service issues
- Review recent pipeline structure changes in CloudTrail
- Check EventBridge rules if pipeline was expected to trigger automatically

## Phase 2 — Remediate

MUST:
- If IAM permission error: update the pipeline service role with required permissions for all action providers
- If action configuration error: fix the action configuration (repository name, branch, project name, deployment group)
- If downstream service error: investigate the specific service (CodeBuild → B1, CodeDeploy → C1)
- If SUPERSEDED: no action needed — this is expected when newer executions overtake older ones

SHOULD:
- Retry the failed execution: `aws codepipeline retry-stage-execution --pipeline-name <pipeline-name> --stage-name <stage> --pipeline-execution-id <execution-id> --retry-mode FAILED_ACTIONS`
- Enable pipeline execution logging via CloudWatch Events for future monitoring
- Verify all action providers (CodeBuild projects, CodeDeploy applications) still exist

MAY:
- Start a new execution manually: `aws codepipeline start-pipeline-execution --name <pipeline-name>`
- Review and update the pipeline structure if actions reference deleted resources

## Common Issues

- symptoms: "ActionExecution failed — InternalError"
  diagnosis: "Pipeline service role lacks permissions for the action provider."
  resolution: "Update the pipeline IAM role to include permissions for the action type (codebuild:StartBuild, codedeploy:CreateDeployment, etc.)."

- symptoms: "Execution status SUPERSEDED"
  diagnosis: "A newer execution overtook this one at a stage boundary. This is expected behavior."
  resolution: "No action needed. The newer execution contains the latest changes."

- symptoms: "Stage failed — action timed out"
  diagnosis: "The action exceeded its timeout. CodePipeline actions have a default timeout depending on the action type."
  resolution: "Check the downstream service (CodeBuild timeout, CodeDeploy deployment duration). Increase timeouts as needed."

- symptoms: "Pipeline not triggering on source changes"
  diagnosis: "Source detection mechanism (polling, webhook, EventBridge) is misconfigured."
  resolution: "Check source configuration. For CodeCommit, verify EventBridge rule exists. For GitHub, verify CodeStar Connection status."

## Output Format

```yaml
root_cause: "Pipeline execution failure — <specific_cause>"
evidence:
  - type: pipeline_state
    content: "<failed stage and action details>"
  - type: execution_history
    content: "<execution status and error message>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry the failed stage"
  long_term: "Add CloudWatch alarms on pipeline execution failures for early detection"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If IAM permission error: update the pipeline service role | YELLOW |
| If action configuration error: fix the action configuration | YELLOW |
| If downstream service error: investigate the specific service | GREEN |
| If SUPERSEDED: no action needed | GREEN |
| Retry the failed execution | GREEN |
| Enable pipeline execution logging via CloudWatch Events | GREEN |
| Verify all action providers still exist | GREEN |
| Start a new execution manually | GREEN |
| Review and update the pipeline structure | YELLOW |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Pipeline execution details may contain source code references, deployment targets, and environment configurations

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If pipeline service role was modified, revert to the previous IAM policy version
2. If action configuration was changed, restore the previous pipeline definition: `aws codepipeline update-pipeline --pipeline <previous-definition>`
3. If a pipeline structure was updated, revert to the previous version using the pipeline definition backup
4. Verify rollback by confirming `aws codepipeline get-pipeline` shows expected configuration

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
