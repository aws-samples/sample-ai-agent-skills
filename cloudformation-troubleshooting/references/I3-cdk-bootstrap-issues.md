---
title: "I3 — CDK Bootstrap Issues"
description: "Diagnose and resolve CDK bootstrap environment problems"
status: active
severity: MEDIUM
triggers:
  - "bootstrap"
  - "CDKToolkit"
  - "bootstrap stack"
  - "StagingBucket"
  - "bootstrap version"
  - "Has the environment been bootstrapped"
owner: devops-agent
objective: "Fix CDK bootstrap issues to enable CDK deployments in the target environment"
context: "CDK bootstrap creates a CDKToolkit stack with an S3 bucket, ECR repository, and IAM roles needed for CDK deployments. Bootstrap issues include: environment not bootstrapped, bootstrap version too old, bootstrap stack corrupted, cross-account trust not configured, and S3 bucket permissions. The bootstrap stack must exist in every account/region where CDK deploys."
---

## Phase 1 — Triage

MUST:
- Check if the bootstrap stack exists: `aws cloudformation describe-stacks --stack-name CDKToolkit`
- Check the bootstrap version: `aws cloudformation describe-stacks --stack-name CDKToolkit --query 'Stacks[0].Outputs[?OutputKey==`BootstrapVersion`].OutputValue'`
- Check the error message for the specific bootstrap requirement
- Verify the S3 staging bucket exists and is accessible

SHOULD:
- Check the bootstrap stack status for failures: `aws cloudformation describe-stack-events --stack-name CDKToolkit`
- Verify IAM roles created by bootstrap: `aws iam get-role --role-name cdk-hnb659fds-deploy-role-<account>-<region>`
- Check if the bootstrap was done with the correct trust policy for cross-account deployments

MAY:
- Check if a custom bootstrap template was used
- Verify ECR repository exists for Docker assets: `aws ecr describe-repositories --repository-names cdk-hnb659fds-container-assets-<account>-<region>`

## Phase 2 — Remediate

MUST:
- If not bootstrapped: `cdk bootstrap aws://<account>/<region>`
- If bootstrap version is too old: `cdk bootstrap aws://<account>/<region>` (re-running upgrades)
- If bootstrap stack is corrupted: delete and re-bootstrap: `aws cloudformation delete-stack --stack-name CDKToolkit` then `cdk bootstrap`
- For cross-account: `cdk bootstrap aws://<target-account>/<region> --trust <deploying-account>`

SHOULD:
- Use a consistent bootstrap template across all environments
- Pin the bootstrap version in CI/CD pipelines
- Use `--cloudformation-execution-policies` to restrict bootstrap role permissions

MAY:
- Customize the bootstrap template for organizational requirements
- Use `cdk bootstrap --show-template` to review the template before deploying

## Common Issues

- symptoms: "'Has the environment been bootstrapped?' error during cdk deploy"
  diagnosis: "The target account/region has not been bootstrapped."
  resolution: "Run cdk bootstrap aws://<account>/<region>."

- symptoms: "Bootstrap version mismatch — requires version N but found M"
  diagnosis: "The CDK app requires a newer bootstrap version than what's deployed."
  resolution: "Re-run cdk bootstrap to upgrade the bootstrap stack."

- symptoms: "Access denied to staging bucket during cdk deploy"
  diagnosis: "IAM permissions on the bootstrap S3 bucket are incorrect."
  resolution: "Re-bootstrap or fix the bucket policy. Check the deploy role permissions."

## Output Format

```yaml
root_cause: "bootstrap_issue — <not_bootstrapped|version_mismatch|corrupted|permissions>"
evidence:
  - type: bootstrap_stack
    content: "<CDKToolkit stack status and version>"
  - type: error_message
    content: "<specific bootstrap error>"
severity: MEDIUM
mitigation:
  immediate: "Bootstrap or re-bootstrap the environment"
  long_term: "Automate bootstrap as part of account provisioning"
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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
