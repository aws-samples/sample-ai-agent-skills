---
title: "A1 — Installation Failures"
description: "Diagnose and resolve LZA initial installation and bootstrap failures"
status: active
severity: CRITICAL
triggers: ["LZA installation failed", "bootstrap error", "installer pipeline failed", "initial deployment error"]
owner: devops-agent
objective: "Successfully install LZA in the management account and establish the accelerator pipeline"
context: "LZA installation creates the AWSAccelerator-Pipeline, supporting CodeBuild projects, S3 buckets for configuration and state, and IAM roles. Installation failures are typically caused by IAM permissions, prerequisite missing, or CloudFormation stack creation errors."
---

## Phase 1 — Triage

MUST:
- Check installer pipeline status: `aws codepipeline get-pipeline-state --name AWSAccelerator-InstallerStack --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}'`
- Check CloudFormation installer stack: `aws cloudformation describe-stacks --stack-name AWSAccelerator-InstallerStack --query 'Stacks[0].{Status:StackStatus,Reason:StackStatusReason}'`
- Verify prerequisites: Control Tower enabled, Organizations active

SHOULD:
- Check CloudFormation stack events for specific errors: `aws cloudformation describe-stack-events --stack-name AWSAccelerator-InstallerStack --max-items 20`
- Verify the management account has required permissions
- Check if LZA was previously installed (conflicting resources)

MAY:
- Review the installer CloudFormation template parameters
- Check service quotas for CodePipeline, CodeBuild, S3, and IAM

## Phase 2 — Remediate

MUST:
- Fix IAM permissions — installer requires AdministratorAccess in the management account
- Ensure Control Tower is set up with a landing zone
- Resolve CloudFormation stack creation errors

SHOULD:
- Verify the S3 configuration bucket was created successfully
- Check CodePipeline and CodeBuild service-linked roles exist
- Ensure no conflicting resources from a previous installation

MAY:
- Clean up partial installation resources before retrying
- Use CloudFormation drift detection to check for manual changes

## Safety Ratings

```
safety_ratings:
  - "cloudformation describe-stacks: GREEN — read-only stack status"
  - "cloudformation describe-stack-events: GREEN — read-only event history"
  - "LZA installation: RED — creates extensive infrastructure in management account"
  - "codepipeline get-pipeline-state: GREEN — read-only pipeline status"
```

## Escalation Conditions

- Installation fails with Control Tower API errors
- CloudFormation stack creation fails with internal service errors
- IAM permissions cannot be granted due to SCP restrictions in the management account
- Previous installation left orphaned resources that conflict

## Data Sensitivity

- CloudFormation stack parameters: HIGH — may contain configuration values
- Management account IAM roles: HIGH — broad permissions for LZA operations
- S3 configuration bucket: MEDIUM — contains LZA configuration files
- CloudFormation stack events: MEDIUM — reveals resource creation details

## Prohibited Actions

- NEVER install LZA in a non-management account
- NEVER install LZA without Control Tower being set up first
- NEVER delete CloudFormation stacks from a partial installation without understanding dependencies
- NEVER use root account credentials for LZA installation

## Phase 3 — Rollback

1. If installation failed partially, check which CloudFormation stacks were created
2. Delete installer stack if it's in ROLLBACK_COMPLETE: `aws cloudformation delete-stack --stack-name AWSAccelerator-InstallerStack`
3. Clean up S3 buckets created by the installer (empty them first)
4. Remove IAM roles created by the installer if they conflict with a retry
5. Document the installation failure and all cleanup actions

## Common Issues

- symptoms: "InstallerStack fails with 'Resource limit exceeded'"
  diagnosis: "Service quota for CloudFormation stacks, IAM roles, or S3 buckets reached."
  resolution: "Check service quotas. Request increases for the specific resource type."

- symptoms: "Pipeline creation fails with 'CodePipeline service role error'"
  diagnosis: "CodePipeline service-linked role does not exist or lacks permissions."
  resolution: "Create the service-linked role: `aws iam create-service-linked-role --aws-service-name codepipeline.amazonaws.com`"

## Output Format

```yaml
root_cause: "installation — <specific_cause>"
evidence:
  - type: installation_error
    content: "<CloudFormation stack status and error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix installation blocker"
  long_term: "Document prerequisites and pre-flight checks"
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
