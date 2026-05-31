---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for LZA issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["LZA error", "general LZA issue", "accelerator error", "LZA not working"]
owner: devops-agent
objective: "Systematically diagnose and resolve general LZA issues"
context: "This runbook covers issues that do not fit neatly into specific categories. It provides a systematic approach to diagnosing any LZA problem by checking common failure points in order of likelihood."
---

## Phase 1 — Triage

MUST:
- Check pipeline status: `aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}'`
- List failed CloudFormation stacks: `aws cloudformation list-stacks --stack-status-filter CREATE_FAILED UPDATE_FAILED ROLLBACK_COMPLETE --query 'StackSummaries[?contains(StackName,`AWSAccelerator`)]'`
- Verify configuration files in S3: `aws s3 ls s3://aws-accelerator-config-<account-id>-<region>/`

SHOULD:
- Check recent pipeline executions: `aws codepipeline list-pipeline-executions --pipeline-name AWSAccelerator-Pipeline --max-items 5`
- Review CloudTrail for recent errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudformation.amazonaws.com --max-results 20`
- Check Organizations status: `aws organizations describe-organization`

MAY:
- Check AWS service health dashboard for outages
- Review IAM policies for recent changes
- Check service quotas across accounts

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Follow the appropriate specific runbook (A1–G1) if the issue matches a known category
- Verify the fix resolves the issue

SHOULD:
- Document the root cause and resolution
- Check if the issue affects other accounts or regions
- Review recent configuration or infrastructure changes

MAY:
- Implement monitoring for LZA pipeline health
- Set up CloudWatch alarms for CloudFormation stack failures
- Create a dashboard for LZA operational metrics

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "cloudformation list-stacks: GREEN — read-only stack listing"
  - "s3 ls: GREEN — read-only bucket listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook
- Multiple LZA components fail simultaneously
- Error messages reference internal service errors
- Issue persists after following all relevant specific runbooks

## Data Sensitivity

- `aws codepipeline get-pipeline-state`: LOW — pipeline stage status
- `aws cloudformation list-stacks`: LOW — stack names and status
- `aws s3 ls`: MEDIUM — configuration file listing
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principals and actions

## Prohibited Actions

- NEVER delete LZA infrastructure components as a troubleshooting step
- NEVER modify CloudFormation stacks directly
- NEVER make manual changes to LZA-managed resources
- NEVER assume the issue is a service bug without exhausting configuration checks

## Phase 3 — Rollback

1. Revert any configuration changes made during troubleshooting
2. If the pipeline was manually triggered, wait for completion before further action
3. If IAM policies were modified, revert to previous versions
4. Document all troubleshooting steps and outcomes
5. Notify stakeholders about any operational impact

## Common Issues

- symptoms: "Pipeline was working but suddenly fails at every stage"
  diagnosis: "IAM role or policy change affecting LZA execution roles."
  resolution: "Check recent IAM changes. Verify LZA roles exist with correct permissions."

- symptoms: "LZA works in some regions but not others"
  diagnosis: "Region-specific issues: service availability, quotas, or configuration targeting."
  resolution: "Check enabled regions in global-config.yaml. Verify service availability in failing regions."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified root cause"
  long_term: "Implement comprehensive LZA monitoring"
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
