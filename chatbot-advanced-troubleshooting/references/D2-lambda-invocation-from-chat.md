---
title: "D2 — Lambda Invocation From Chat"
description: "Diagnose Lambda function invocation from chat failures including permissions, response formatting, and timeout issues"
status: active
severity: MEDIUM
triggers: ["Lambda invocation failed from chat", "function error in chat", "Lambda timeout from chat", "Lambda response not showing"]
owner: devops-agent
objective: "Resolve Lambda function invocation failures from AWS Chatbot"
context: "Lambda invocation from chat uses the channel IAM role to call lambda:InvokeFunction. Responses must be formatted for the target platform. Large responses are truncated."
---

## Phase 1 — Triage

MUST:
- Check channel IAM role for Lambda permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names lambda:InvokeFunction`
- Verify the Lambda function exists: `aws lambda get-function --function-name <function-name>`
- Check guardrails allow Lambda invocation

SHOULD:
- Review Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <timestamp>`
- Check Lambda function timeout and memory configuration
- Verify the function returns a response formatted for chat

MAY:
- Test the function directly: `aws lambda invoke --function-name <name> --payload '{}' output.json`
- Check Lambda function VPC configuration if it needs network access

## Phase 2 — Remediate

MUST:
- Grant the channel IAM role lambda:InvokeFunction permission for the specific function
- Fix Lambda function errors (runtime, timeout, memory)
- Ensure guardrails allow Lambda invocation

SHOULD:
- Format Lambda responses for the target platform (Slack Block Kit / Teams Adaptive Cards)
- Keep responses concise to avoid truncation
- Set appropriate Lambda timeout (consider chat response time expectations)

MAY:
- Implement response pagination for large outputs
- Use S3 presigned URLs for large data in responses
- Add error handling in the Lambda function for chat-specific scenarios

## Safety Ratings

```
safety_ratings:
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "lambda get-function: GREEN — read-only function metadata"
  - "lambda invoke: YELLOW — executes function code"
  - "logs filter-log-events: GREEN — read-only log access"
```

## Escalation Conditions

- Lambda function invocation causes unintended side effects
- Function timeout affects chat user experience
- Lambda function requires VPC access that is not configured
- Response formatting issues affect multiple chat platforms

## Data Sensitivity

- `aws iam simulate-principal-policy`: LOW — returns allow/deny results
- `aws lambda get-function`: LOW — returns function metadata
- `aws lambda invoke`: MEDIUM to HIGH — depends on function behavior
- `aws logs filter-log-events`: HIGH — may contain application data

## Prohibited Actions

- NEVER invoke Lambda functions that perform destructive operations without approval
- NEVER grant lambda:InvokeFunction with wildcard function ARNs
- NEVER expose sensitive data in Lambda responses displayed in chat
- NEVER set Lambda timeout longer than chat response timeout

## Phase 3 — Rollback

1. If the Lambda function made unwanted changes, revert them based on function logic
2. If IAM permissions were over-granted, revert to previous policy
3. If the function was modified, deploy the previous version
4. Verify the function works correctly after changes
5. Document the invocation failure and resolution

## Common Issues

- symptoms: "Lambda invocation returns 'access denied' in chat"
  diagnosis: "The channel IAM role lacks lambda:InvokeFunction permission for the function."
  resolution: "Add lambda:InvokeFunction permission for the specific function ARN to the channel role."

- symptoms: "Lambda function times out when invoked from chat"
  diagnosis: "The function execution time exceeds the configured timeout."
  resolution: "Optimize function code. Increase Lambda timeout. Consider async invocation for long operations."

- symptoms: "Lambda response appears garbled or truncated in chat"
  diagnosis: "The response is not formatted for the target platform or exceeds size limits."
  resolution: "Format responses for Slack (Block Kit) or Teams (Adaptive Cards). Keep responses under size limits."

## Output Format

```yaml
root_cause: "lambda_invocation_chat — <specific_cause>"
evidence:
  - type: lambda_error
    content: "<invocation failure details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda invocation issue"
  long_term: "Implement proper response formatting and error handling"
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
