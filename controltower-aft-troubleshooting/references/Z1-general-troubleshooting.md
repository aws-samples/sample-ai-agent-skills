---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for AFT issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["AFT error", "general AFT issue", "unknown AFT error", "AFT not working"]
owner: devops-agent
objective: "Systematically diagnose and resolve general AFT issues"
context: "This runbook covers issues that do not fit neatly into specific categories. It provides a systematic approach to diagnosing any AFT problem by checking common failure points in order of likelihood."
---

## Phase 1 — Triage

MUST:
- List all AFT pipelines and their status: `aws codepipeline list-pipelines --query 'pipelines[?contains(name,`aft`)].name'`
- Check each pipeline's latest execution: `aws codepipeline get-pipeline-state --name <pipeline-name>`
- Scan DynamoDB for account request status: `aws dynamodb scan --table-name aft-request --select COUNT`

SHOULD:
- Check AFT Lambda functions: `aws lambda list-functions --query 'Functions[?contains(FunctionName,`aft`)].{Name:FunctionName,LastModified:LastModified}'`
- Review CloudTrail for recent AFT-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=controltower.amazonaws.com --max-results 20`
- Verify Terraform state backend: `aws s3 ls s3://aft-backend-<account-id>-<region>/`

MAY:
- Check AWS service health dashboard for Control Tower or CodePipeline outages
- Review IAM policies for recent changes affecting AFT roles
- Check service quotas for CodePipeline, CodeBuild, and Lambda

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Follow the appropriate specific runbook (A1–G1) if the issue matches a known category
- Verify the fix resolves the issue

SHOULD:
- Document the root cause and resolution
- Check if the issue affects other AFT operations
- Review recent infrastructure or configuration changes

MAY:
- Implement monitoring for AFT pipeline health
- Set up CloudWatch alarms for AFT Lambda function errors
- Create a dashboard for AFT operational metrics

## Safety Ratings

```
safety_ratings:
  - "codepipeline list-pipelines: GREEN — read-only enumeration"
  - "dynamodb scan: GREEN — read-only table scan"
  - "lambda list-functions: GREEN — read-only function listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook
- Multiple AFT components fail simultaneously
- Error messages reference internal service errors
- Issue persists after following all relevant specific runbooks

## Data Sensitivity

- `aws codepipeline list-pipelines`: LOW — pipeline names only
- `aws dynamodb scan --table-name aft-request`: HIGH — contains account request details
- `aws lambda list-functions`: LOW — function metadata only
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principals and actions

## Prohibited Actions

- NEVER delete AFT infrastructure components as a troubleshooting step
- NEVER modify DynamoDB tables directly
- NEVER bypass AFT to make changes in managed accounts
- NEVER assume the issue is a service bug without exhausting configuration checks

## Phase 3 — Rollback

1. Revert any configuration changes made during troubleshooting
2. If pipelines were manually triggered, wait for completion before further action
3. If IAM policies were modified, revert to previous versions
4. Document all troubleshooting steps and outcomes
5. Notify stakeholders about any operational impact

## Common Issues

- symptoms: "All AFT operations suddenly stopped working"
  diagnosis: "IAM role or policy change affecting AFT service roles."
  resolution: "Check recent IAM changes. Verify AFT roles exist and have correct permissions."

- symptoms: "AFT works for some accounts but not others"
  diagnosis: "Account-specific issues: missing AWSAFTExecution role, SCP restrictions, or OU configuration."
  resolution: "Compare working and failing accounts. Check role existence and SCP policies in the target OU."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified root cause"
  long_term: "Implement comprehensive AFT monitoring"
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
