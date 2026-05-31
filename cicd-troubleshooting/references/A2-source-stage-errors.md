---
title: "A2 — Source Stage Errors"
description: "Diagnose and resolve CodePipeline source stage failures"
status: active
severity: HIGH
triggers:
  - "source stage failed"
  - "source action failed"
  - "repository not found"
  - "branch not found"
  - "connection failed"
  - "OAuth token"
owner: devops-agent
objective: "Identify why the source stage failed to pull code and resolve the issue"
context: "The source stage connects CodePipeline to a code repository (CodeCommit, GitHub, S3, ECR). Failures typically stem from connection issues, repository/branch misconfiguration, authentication problems, or webhook/polling misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check pipeline state for source stage status: `aws codepipeline get-pipeline-state --name <pipeline-name>`
- Get source action configuration: `aws codepipeline get-pipeline --name <pipeline-name> --query 'pipeline.stages[?name==`Source`]'`
- For CodeCommit: verify repository exists: `aws codecommit get-repository --repository-name <repo>`
- For CodeCommit: verify branch exists: `aws codecommit get-branch --repository-name <repo> --branch-name <branch>`

SHOULD:
- For GitHub (CodeStar Connection): check connection status: `aws codestar-connections list-connections`
- For S3 source: verify bucket and object key exist: `aws s3api head-object --bucket <bucket> --key <key>`
- For ECR source: verify image exists: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check if source detection is polling or event-based: `aws codepipeline get-pipeline --query 'pipeline.stages[0].actions[0].configuration'`

MAY:
- For GitHub: verify OAuth token or CodeStar Connection has repository access
- Check EventBridge rules for CodeCommit source: `aws events list-rules --name-prefix "codepipeline"`
- Check CloudTrail for source action errors

## Phase 2 — Remediate

MUST:
- If repository not found: update the source action with the correct repository name
- If branch not found: update the source action with the correct branch name
- If connection failed (GitHub): reauthorize the CodeStar Connection in the console
- If S3 object not found: verify the S3 key path and ensure the object exists

SHOULD:
- For CodeCommit: switch from polling to EventBridge for faster detection
- For GitHub: use CodeStar Connections instead of OAuth tokens (more reliable, supports GitHub Apps)
- Verify the pipeline service role has permissions to access the source provider

MAY:
- Test source connectivity manually before retrying the pipeline
- Update webhook configuration if using legacy webhook-based triggers

## Common Issues

- symptoms: "Source action failed — Could not find repository"
  diagnosis: "Repository name in the source action does not match an existing repository."
  resolution: "Verify repository name and region. CodeCommit repositories are region-specific."

- symptoms: "Source action failed — Branch not found"
  diagnosis: "The specified branch does not exist in the repository."
  resolution: "Check branch name for typos. Verify the branch exists: `aws codecommit list-branches --repository-name <repo>`."

- symptoms: "GitHub connection status PENDING or ERROR"
  diagnosis: "CodeStar Connection was not completed or authorization expired."
  resolution: "Complete the connection setup in the AWS Console under Developer Tools > Connections."

- symptoms: "Source stage not triggering on new commits"
  diagnosis: "Change detection mechanism is misconfigured or disabled."
  resolution: "For CodeCommit: verify EventBridge rule. For GitHub: verify CodeStar Connection webhook. For S3: verify CloudTrail + EventBridge."

## Output Format

```yaml
root_cause: "Source stage error — <specific_cause>"
evidence:
  - type: pipeline_state
    content: "<source action error details>"
  - type: source_config
    content: "<repository, branch, connection status>"
severity: HIGH
mitigation:
  immediate: "Fix source configuration and retry pipeline"
  long_term: "Use CodeStar Connections for GitHub, EventBridge for CodeCommit"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If repository not found: update the source action with correct repository name | YELLOW |
| If branch not found: update the source action with correct branch name | YELLOW |
| If connection failed (GitHub): reauthorize the CodeStar Connection | GREEN |
| If S3 object not found: verify the S3 key path | GREEN |
| For CodeCommit: switch from polling to EventBridge | GREEN |
| For GitHub: use CodeStar Connections instead of OAuth tokens | GREEN |
| Verify the pipeline service role has permissions to access the source provider | YELLOW |
| Test source connectivity manually before retrying | GREEN |
| Update webhook configuration | YELLOW |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Source stage configurations contain repository URLs, branch names, and connection credentials

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If source action configuration was changed, restore the previous repository/branch settings in the pipeline definition
2. If CodeStar Connection was reauthorized, verify it still has access to the correct repositories
3. If webhook configuration was updated, restore the previous webhook settings
4. Verify rollback by confirming `aws codepipeline get-pipeline` source stage matches expected configuration

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
