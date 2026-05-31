---
title: "A2 — Pipeline Execution Errors"
description: "Diagnose and resolve AWSAccelerator-Pipeline execution failures"
status: active
severity: HIGH
triggers: ["pipeline failed", "AWSAccelerator-Pipeline error", "pipeline stage failed", "accelerator pipeline stuck"]
owner: devops-agent
objective: "Identify and resolve the specific stage causing LZA pipeline failures"
context: "The AWSAccelerator-Pipeline has multiple stages that deploy CloudFormation stacks across accounts and regions. Stages include Source, Build, Review, Prepare, Accounts, Organizations, SecurityAudit, Network, Security, Operations, and Finalize. Each stage can fail independently."
---

## Phase 1 — Triage

MUST:
- Get pipeline execution state: `aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}'`
- List recent executions: `aws codepipeline list-pipeline-executions --pipeline-name AWSAccelerator-Pipeline --max-items 5`
- Identify the failed stage and get CodeBuild build ID

SHOULD:
- Get build details: `aws codebuild batch-get-builds --ids <build-id> --query 'builds[0].{Status:buildStatus,Phase:currentPhase}'`
- Review CodeBuild logs for the failed stage
- Check if the failure is in CDK synth or CloudFormation deployment

MAY:
- Compare with the last successful pipeline execution
- Check for manual approval stages that may be blocking

## Phase 2 — Remediate

MUST:
- Fix the root cause identified in the failed stage's CodeBuild logs
- For CDK synth failures: fix configuration file errors
- For CloudFormation failures: fix resource-level issues

SHOULD:
- Retry the pipeline after fixing the issue (pipeline will resume from the failed stage)
- Verify configuration files are valid YAML with correct schema
- Check for resource conflicts in target accounts

MAY:
- Enable detailed CloudFormation logging for the failing stacks
- Review the CDK diff output to understand what changes are being attempted

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "codepipeline list-pipeline-executions: GREEN — read-only history"
  - "codebuild batch-get-builds: GREEN — read-only build details"
  - "Pipeline re-execution: RED — deploys infrastructure across multiple accounts"
```

## Escalation Conditions

- Pipeline fails at the same stage repeatedly after configuration fixes
- Multiple stages fail simultaneously suggesting a systemic issue
- Pipeline execution hangs without progressing or failing
- Manual approval stage requires authorization from unavailable approver

## Data Sensitivity

- `aws codepipeline get-pipeline-state`: LOW — stage status only
- CodeBuild logs: HIGH — contain CDK output, CloudFormation details, and resource information
- Pipeline execution details: MEDIUM — reveals deployment workflow
- Configuration files in S3: HIGH — contain organization structure and security configuration

## Prohibited Actions

- NEVER skip the manual approval stage without reviewing the CDK diff
- NEVER re-run the pipeline without understanding why it failed
- NEVER modify pipeline configuration outside of LZA
- NEVER cancel a running pipeline execution unless absolutely necessary

## Phase 3 — Rollback

1. If pipeline deployed incorrect changes, revert configuration files and re-run
2. If CloudFormation stacks are in a failed state, check if they can be updated or need deletion
3. If the pipeline is stuck, check for manual approval or CodeBuild timeout
4. Document the pipeline failure, affected stages, and resolution
5. Verify all accounts and regions are in a consistent state after recovery

## Common Issues

- symptoms: "Pipeline fails at the Network stage"
  diagnosis: "network-config.yaml has errors — CIDR conflicts, invalid TGW references, or missing VPC dependencies."
  resolution: "Review network-config.yaml. Check CIDR ranges don't overlap. Verify TGW and VPC references exist."

- symptoms: "Pipeline fails at the Accounts stage"
  diagnosis: "accounts-config.yaml references non-existent OUs or has invalid account configurations."
  resolution: "Verify OU names match organization-config.yaml. Check account email uniqueness."

## Output Format

```yaml
root_cause: "pipeline_execution — <specific_cause>"
evidence:
  - type: pipeline_error
    content: "<failed stage, CodeBuild logs, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and retry pipeline"
  long_term: "Implement pre-pipeline configuration validation"
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
