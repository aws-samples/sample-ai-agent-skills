---
title: "F1 — Custom Action Errors"
description: "Diagnose custom automation action failures including aws:executeScript, aws:invokeLambdaFunction, and custom step type errors"
status: active
severity: MEDIUM
triggers: ["custom action failed", "executeScript error", "Lambda invocation failed", "script runtime error", "custom step error"]
owner: devops-agent
objective: "Resolve custom action execution errors in automation documents"
context: "Custom actions include aws:executeScript (inline Python/PowerShell), aws:invokeLambdaFunction, and aws:executeAwsApi. Each has different runtime environments, timeout limits, and error handling behaviors."
---

## Phase 1 — Triage

MUST:
- Get step execution details: `aws ssm describe-automation-step-executions --automation-execution-id <execution-id> --filters Key=StepName,Values=<step-name>`
- Check the step failure message and output
- Review the step definition in the document: `aws ssm get-document --name <document-name>`

SHOULD:
- For aws:executeScript: check the script runtime (python3.8, PowerShell) and handler function
- For aws:invokeLambdaFunction: check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <timestamp>`
- For aws:executeAwsApi: verify the API call parameters and permissions

MAY:
- Test the script independently outside of SSM Automation
- Check Lambda function configuration and timeout settings
- Review IAM permissions for the specific API calls

## Phase 2 — Remediate

MUST:
- Fix script syntax or runtime errors for aws:executeScript
- Fix Lambda function errors or timeout for aws:invokeLambdaFunction
- Correct API parameters for aws:executeAwsApi

SHOULD:
- Ensure aws:executeScript returns output in the correct format (dict with string values)
- Set appropriate timeout for Lambda invocations (Lambda timeout + buffer)
- Add error handling within custom scripts

MAY:
- Add logging to custom scripts for better debugging
- Implement retry logic for transient failures
- Use aws:executeScript for complex logic instead of chaining multiple aws:executeAwsApi steps

## Safety Ratings

```
safety_ratings:
  - "describe-automation-step-executions: GREEN — read-only step details"
  - "get-document: GREEN — read-only document content"
  - "logs filter-log-events: GREEN — read-only log access"
  - "update-document: YELLOW — modifies automation document"
```

## Escalation Conditions

- Custom script executes destructive actions and fails mid-operation
- Lambda function timeout causes automation to hang
- aws:executeAwsApi calls undocumented API behavior
- Script runtime environment limitations prevent required functionality

## Data Sensitivity

- `aws ssm describe-automation-step-executions`: MEDIUM — may contain step outputs
- `aws ssm get-document`: MEDIUM — contains script source code
- `aws logs filter-log-events`: HIGH — may contain application data and secrets in logs
- `aws lambda get-function`: LOW — returns function metadata

## Prohibited Actions

- NEVER include secrets or credentials in aws:executeScript inline code
- NEVER set unlimited timeout for Lambda invocations
- NEVER use aws:executeAwsApi for actions that require complex error handling
- NEVER deploy untested custom scripts to production automation documents

## Phase 3 — Rollback

1. If a custom action modified resources before failing, identify the changes
2. For aws:executeScript, review the script logic to determine what was changed
3. For aws:invokeLambdaFunction, check Lambda execution logs for completed actions
4. Manually revert any partial changes made by the failed custom action
5. Update the document with fixed custom action code before re-executing

## Common Issues

- symptoms: "aws:executeScript fails with ScriptFailed and no useful error message"
  diagnosis: "The Python/PowerShell script has a runtime error or returns output in wrong format."
  resolution: "Script must return a dict with string values. Add try/except blocks. Test script independently."

- symptoms: "aws:invokeLambdaFunction times out"
  diagnosis: "Lambda function execution exceeds the step timeout or Lambda's own timeout."
  resolution: "Increase Lambda timeout. Set step timeoutSeconds to Lambda timeout + 30 seconds buffer."

- symptoms: "aws:executeAwsApi returns unexpected output format"
  diagnosis: "The API response structure does not match the expected output selector."
  resolution: "Check the AWS API documentation for exact response structure. Update the Selector in the step definition."

## Output Format

```yaml
root_cause: "custom_action — <step_type> — <specific_cause>"
evidence:
  - type: custom_action_error
    content: "<step failure message and script/function details>"
severity: MEDIUM
mitigation:
  immediate: "Fix custom action code"
  long_term: "Add testing and error handling to custom actions"
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
