---
title: "E1 — CodePipeline Errors"
description: "Diagnose and resolve CodePipeline execution errors in AFT infrastructure"
status: active
severity: HIGH
triggers: ["CodePipeline failed", "pipeline execution error", "pipeline stage failed", "source action failed"]
owner: devops-agent
objective: "Resolve CodePipeline execution failures that block AFT operations"
context: "AFT relies on CodePipeline for orchestrating account provisioning and customization workflows. CodePipeline failures can occur at source, build, or deploy stages. Most AFT pipeline issues are actually CodePipeline configuration or permission problems."
---

## Phase 1 — Triage

MUST:
- Get pipeline execution history: `aws codepipeline list-pipeline-executions --pipeline-name <name> --max-items 5`
- Get failed stage details: `aws codepipeline get-pipeline-state --name <name>`
- Check pipeline execution details: `aws codepipeline get-pipeline-execution --pipeline-name <name> --pipeline-execution-id <id>`

SHOULD:
- Identify the specific failed action within the stage
- Check CodePipeline service role permissions
- Review source repository for recent changes that triggered the pipeline

MAY:
- Check for pipeline webhook or trigger configuration issues
- Review CloudTrail for CodePipeline API errors

## Phase 2 — Remediate

MUST:
- Fix the specific stage/action failure (source access, build errors, deploy failures)
- Ensure CodePipeline service role has required permissions for all stages
- Retry the failed stage: `aws codepipeline retry-stage-execution --pipeline-name <name> --stage-name <stage> --pipeline-execution-id <id> --retry-mode FAILED_ACTIONS`

SHOULD:
- Verify source repository connectivity and branch configuration
- Check artifact store (S3 bucket) accessibility
- Ensure no concurrent executions are conflicting

MAY:
- Enable pipeline execution notifications
- Review and optimize pipeline stage configuration

## Safety Ratings

```
safety_ratings:
  - "codepipeline list-pipeline-executions: GREEN — read-only history"
  - "codepipeline get-pipeline-state: GREEN — read-only status"
  - "codepipeline retry-stage-execution: YELLOW — re-executes failed stage"
  - "codepipeline start-pipeline-execution: YELLOW — triggers full pipeline run"
```

## Escalation Conditions

- Pipeline service role permissions cannot be modified due to SCP restrictions
- Pipeline artifact store S3 bucket is inaccessible or corrupted
- Pipeline configuration is managed by Terraform and manual fixes would cause drift
- Multiple AFT pipelines fail simultaneously

## Data Sensitivity

- `aws codepipeline get-pipeline-execution`: MEDIUM — execution details and artifact references
- Pipeline configuration: MEDIUM — reveals infrastructure workflow
- Pipeline artifacts in S3: HIGH — may contain Terraform plans and account details
- CodePipeline service role: MEDIUM — reveals permission scope

## Prohibited Actions

- NEVER delete AFT CodePipeline pipelines manually
- NEVER modify pipeline configuration outside of Terraform
- NEVER retry pipelines without understanding the failure cause
- NEVER grant CodePipeline service role AdministratorAccess

## Phase 3 — Rollback

1. If pipeline retry caused additional issues, wait for completion before investigating
2. If pipeline configuration was modified manually, run AFT Terraform to restore
3. If source repository changes caused the failure, revert the commit
4. Document pipeline failures and resolutions
5. Verify downstream operations are not affected

## Common Issues

- symptoms: "Pipeline source stage fails with 'Could not access the CodeCommit repository'"
  diagnosis: "CodePipeline service role lacks codecommit:GitPull permission or repository was renamed."
  resolution: "Check service role permissions. Verify repository name matches pipeline source configuration."

- symptoms: "Pipeline execution superseded by newer execution"
  diagnosis: "Multiple commits triggered concurrent executions; newer one superseded the older."
  resolution: "This is normal behavior. Check the latest execution status. Configure execution mode if needed."

## Output Format

```yaml
root_cause: "codepipeline — <specific_cause>"
evidence:
  - type: pipeline_error
    content: "<stage, action, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix pipeline stage failure and retry"
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
