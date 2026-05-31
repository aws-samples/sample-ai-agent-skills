---
title: "A1 — Automation Execution Failures"
description: "Diagnose SSM Automation execution failures including status, permissions, and parameter errors"
status: active
severity: HIGH
triggers: ["automation failed", "execution failed", "automation not running", "StartAutomationExecution error"]
owner: devops-agent
objective: "Resolve automation execution failures and restore successful execution"
context: "SSM Automation executions can fail due to IAM permissions, invalid parameters, document errors, or service limits. Executions use $DEFAULT document version unless specified."
---

## Phase 1 — Triage

MUST:
- List recent failed executions: `aws ssm describe-automation-executions --filters Key=ExecutionStatus,Values=Failed --max-results 10`
- Get execution details: `aws ssm get-automation-execution --automation-execution-id <execution-id>`
- Check the FailureMessage and ExecutionStatus fields in the response

SHOULD:
- Review step-level failures: `aws ssm describe-automation-step-executions --automation-execution-id <execution-id>`
- Verify the document exists and version: `aws ssm describe-document --name <document-name>`
- Check CloudTrail for StartAutomationExecution events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StartAutomationExecution --max-results 10`

MAY:
- Check service quotas: `aws service-quotas get-service-quota --service-code ssm --quota-code L-09101E40`
- Review CloudWatch metrics for SSM Automation

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions if AccessDeniedException
- Correct invalid parameters if ValidationException
- Update document version if document errors

SHOULD:
- Verify the AssumeRole parameter points to a valid IAM role with correct trust policy
- Ensure all required document parameters are provided with valid values
- Check that the document version ($DEFAULT or specified) is correct

MAY:
- Request service quota increase if hitting concurrent execution limits
- Add CloudWatch alarms for automation failure monitoring

## Safety Ratings

```
safety_ratings:
  - "describe-automation-executions: GREEN — read-only listing of executions"
  - "get-automation-execution: GREEN — read-only execution details"
  - "start-automation-execution: YELLOW — initiates automation that may modify resources"
  - "stop-automation-execution: RED — cancels running automation, may leave resources in partial state"
```

## Escalation Conditions

- Execution fails with InternalServerError repeatedly after parameter and permission fixes
- Concurrent execution limit reached and quota increase is pending
- Execution failure affects production change management pipeline
- Service role permissions cannot be determined from error messages

## Data Sensitivity

- `aws ssm describe-automation-executions`: LOW — returns execution metadata
- `aws ssm get-automation-execution`: MEDIUM — may contain parameter values including secrets
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and actions
- `aws iam get-role`: LOW — returns role metadata and trust policy

## Prohibited Actions

- NEVER stop a running automation execution without explicit approval from the execution owner
- NEVER modify the automation document to bypass permission checks
- NEVER hardcode credentials in automation parameters — use IAM roles or Parameter Store
- NEVER increase MaxConcurrency on production automations without load testing

## Phase 3 — Rollback

1. If an automation was started in error, stop it: `aws ssm stop-automation-execution --automation-execution-id <id>` (with owner approval)
2. Review any resources modified by completed steps before the failure
3. Manually revert resource changes if the automation lacks rollback steps
4. If IAM role was modified, restore previous policy: `aws iam put-role-policy --role-name <name> --policy-name <policy> --policy-document file://original-policy.json`
5. Document the failure and rollback actions for post-incident review

## Common Issues

- symptoms: "Execution fails immediately with AccessDeniedException"
  diagnosis: "The AssumeRole IAM role lacks required permissions or has an incorrect trust policy."
  resolution: "Verify role trust policy allows ssm.amazonaws.com. Add required permissions for all document steps."

- symptoms: "Execution fails with InvalidParameterValue"
  diagnosis: "A required parameter is missing or has an invalid value type."
  resolution: "Check document parameter definitions. Ensure all required parameters are provided with correct types."

- symptoms: "Execution stuck in InProgress for hours"
  diagnosis: "A step is waiting for a condition or approval, or a long-running step has not completed."
  resolution: "Check step-level status. Look for aws:approve steps waiting for approval or aws:waitForAwsResourceProperty steps."

## Output Format

```yaml
root_cause: "automation_execution — <specific_cause>"
evidence:
  - type: execution_error
    content: "<FailureMessage from get-automation-execution>"
severity: HIGH
mitigation:
  immediate: "Fix execution failure cause"
  long_term: "Add pre-execution validation and monitoring"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
