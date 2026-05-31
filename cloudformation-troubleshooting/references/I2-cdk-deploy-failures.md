---
title: "I2 — CDK Deploy Failures"
description: "Diagnose and resolve CDK deployment failures (CloudFormation-level errors)"
status: active
severity: HIGH
triggers:
  - "cdk deploy"
  - "deploy failed"
  - "CDK deployment error"
  - "stack failed"
  - "CDK rollback"
owner: devops-agent
objective: "Identify whether the failure is in CDK asset publishing or CloudFormation execution, and fix accordingly"
context: "CDK deploy has three phases: (1) synth (template generation), (2) asset publishing (uploading Lambda code, Docker images to S3/ECR), and (3) CloudFormation execution. Deploy failures can occur in any phase. Phase 1 errors are local code issues (see I1). Phase 2 errors are S3/ECR upload issues. Phase 3 errors are CloudFormation stack failures (see A1-A4)."
---

## Phase 1 — Triage

MUST:
- Run deploy with verbose output: `cdk deploy --verbose 2>&1`
- Determine which phase failed: synth, asset publishing, or CloudFormation execution
- If CloudFormation execution failed: check stack events: `aws cloudformation describe-stack-events --stack-name <stack>`
- If asset publishing failed: check S3 bucket and ECR repository access

SHOULD:
- Check the CDK output for the specific error message
- Verify AWS credentials and region: `aws sts get-caller-identity`
- Check if the bootstrap stack is healthy (see I3)
- Review the CloudFormation stack status: `aws cloudformation describe-stacks --stack-name <stack>`

MAY:
- Check if the deployment role has sufficient permissions
- Review the generated template in `cdk.out/` for issues
- Check if hotswap deployment was used and caused issues

## Phase 2 — Remediate

MUST:
- For synth failures: see I1
- For asset publishing failures: verify S3 bucket permissions and ECR repository access
- For CloudFormation failures: see the appropriate A-category runbook based on stack status
- Fix the root cause and retry: `cdk deploy`

SHOULD:
- Use `cdk diff` before deploy to preview changes
- Use `--require-approval broadening` to review security-sensitive changes
- Check if `--force` is needed for stuck deployments

MAY:
- Use `cdk deploy --hotswap` for faster Lambda/ECS updates in development (not production)
- Use `cdk deploy --rollback false` to preserve failed state for debugging

## Common Issues

- symptoms: "CDK deploy fails with 'Access Denied' during asset upload"
  diagnosis: "The deploy role cannot write to the CDK staging S3 bucket."
  resolution: "Re-bootstrap the environment or fix the deploy role permissions."

- symptoms: "CDK deploy fails with CloudFormation stack errors"
  diagnosis: "The generated template is valid but CloudFormation can't execute it."
  resolution: "Check stack events for the specific resource failure. See A1-A4 runbooks."

- symptoms: "CDK deploy hangs during 'Publishing assets'"
  diagnosis: "Large assets (Docker images) take time to upload, or network issues."
  resolution: "Check network connectivity. For large Docker images, consider using ECR caching."

## Output Format

```yaml
root_cause: "cdk_deploy_failure — phase: <synth|asset_publishing|cloudformation>: <error>"
evidence:
  - type: deploy_output
    content: "<cdk deploy error output>"
  - type: stack_events
    content: "<CloudFormation stack events if applicable>"
severity: HIGH
mitigation:
  immediate: "Fix the failing phase and retry deployment"
  long_term: "Add cdk diff to CI/CD pipeline and use change set approval gates"
```


## Safety Ratings
- GREEN: describe-stacks, describe-stack-events, get-template, validate-template
- YELLOW: continue-update-rollback, create-change-set, update-stack
- RED: delete-stack, set-stack-policy with open policy, skip resources during rollback

## Escalation Conditions
- "Stack is in UPDATE_ROLLBACK_FAILED state"
- "Remediation requires skipping resources during rollback"
- "Fix involves modifying stack policy"
- "Nested stack failure in production"

## Rollback
- Pre-change: "Create change set before any update"
- Verification: "Check stack events after operation"
- Revert: "Use rollback triggers or manual rollback"

## Data Sensitivity
- MEDIUM: "Stack templates may contain hardcoded credentials or account IDs"
- MEDIUM: "Stack outputs may expose endpoint URLs and resource IDs"

## Prohibited Actions
- NEVER leave a stack policy open after temporary modification
- NEVER suggest deleting a production stack to fix UPDATE_ROLLBACK_FAILED without exploring ContinueUpdateRollback
- NEVER suggest `--resources-to-skip` without documenting which resources and why
- NEVER suggest modifying CT-managed stacks (AWSControlTower* prefix)

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
  - "NEVER leave stack policy open after temporary modification"
  - "NEVER delete a stack without exploring ContinueUpdateRollback first"
  - "NEVER suggest disabling rollback on production stacks"
