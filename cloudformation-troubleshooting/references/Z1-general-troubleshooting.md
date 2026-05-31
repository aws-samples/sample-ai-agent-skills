---
title: "Z1 — General CloudFormation Troubleshooting (Catch-All)"
description: "Fallback SOP for CloudFormation issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown CloudFormation issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get the stack status: `aws cloudformation describe-stacks --stack-name <stack>`
- Get recent stack events: `aws cloudformation describe-stack-events --stack-name <stack> --max-items 20`
- Get the current template: `aws cloudformation get-template --stack-name <stack>`
- List all resources and their status: `aws cloudformation list-stack-resources --stack-name <stack>`
- Check for drift: `aws cloudformation detect-stack-drift --stack-name <stack>`

SHOULD:
- Check CloudTrail for recent CloudFormation API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudformation.amazonaws.com --max-results 10`
- Validate the template: `aws cloudformation validate-template --template-body file://template.yaml`
- Check stack parameters and outputs: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].{Parameters:Parameters,Outputs:Outputs}'`
- Check if a stack policy exists: `aws cloudformation get-stack-policy --stack-name <stack>`

MAY:
- Check AWS Health Dashboard for CloudFormation service events
- Review CloudWatch metrics for CloudFormation
- Check if the issue is CDK-related (check for CDKMetadata resource)

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Stack in *_FAILED state → Stack Failures (A1-A4)
- Template validation errors → Template Issues (B1-B3)
- Individual resource failures → Resource Errors (C1-C4)
- Out-of-band changes detected → Drift (D1-D3)
- Nested stack involved → Nested Stacks (E1-E2)
- Custom resource involved → Custom Resources (F1-F3)
- StackSet operation → StackSets (G1-G3)
- Change set issues → Change Sets (H1-H2)
- CDK-related → CDK (I1-I3)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all stack configurations systematically
- Review CloudTrail for recent configuration changes
- Check IAM permissions for the CloudFormation execution role
- Verify service quotas in the affected region

SHOULD:
- Check AWS Health Dashboard for service events
- Compare with a known-good stack configuration
- Check if the issue is region-specific

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<stack_failure|template|resource|drift|nested|custom_resource|stackset|changeset|cdk|unknown>"
investigation_path: "stack status → events → template → resources → <domain_classification>"
evidence:
  - type: stack_status
    content: "<stack status and events summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
