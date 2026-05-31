---
title: "A2 — Custom Rule Lambda Errors"
description: "Diagnose Lambda function errors in AWS Config custom rules"
status: active
severity: HIGH
triggers:
  - "custom rule error"
  - "lambda function failed"
  - "custom rule timeout"
  - "lambda invocation error"
  - "custom rule not working"
owner: devops-agent
objective: "Identify and resolve Lambda function errors preventing custom Config rule evaluation"
context: "Custom Config rules invoke Lambda functions for evaluation. Failures can stem from Lambda throttling, timeout, permission issues, malformed evaluation responses, or runtime errors. Lambda CloudWatch logs are the primary diagnostic source."
---

## Phase 1 — Triage

MUST:
- Check rule evaluation status: `aws configservice describe-config-rule-evaluation-status --config-rule-names <rule-name>`
- Get the Lambda function ARN from the rule: `aws configservice describe-config-rules --config-rule-names <rule-name>`
- Check Lambda function configuration: `aws lambda get-function --function-name <function-name>`
- Check Lambda invocation errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check Lambda throttles: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Throttles --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check Lambda CloudWatch logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --limit 50`
- Verify the Lambda execution role has Config permissions: `aws iam get-role --role-name <lambda-role>`
- Check Lambda concurrency settings: `aws lambda get-function-concurrency --function-name <function-name>`

MAY:
- Test the Lambda function with a sample Config event: `aws lambda invoke --function-name <function-name> --payload '<test-event>' output.json`
- Check reserved concurrency limits: `aws lambda get-account-settings`

## Phase 2 — Remediate

MUST:
- Fix Lambda function code errors identified in CloudWatch logs
- Ensure the Lambda execution role has `config:PutEvaluations` permission
- Verify the function returns evaluations in the correct format

SHOULD:
- Increase Lambda timeout if the function is timing out (max 15 minutes)
- Increase Lambda memory if the function is running out of memory
- Add error handling and logging to the Lambda function

MAY:
- Set up Lambda Dead Letter Queue for failed invocations
- Implement retry logic in the Lambda function for transient errors

## Common Issues

- symptoms: "Lambda function times out during evaluation"
  diagnosis: "The function takes too long to evaluate resources, possibly due to API calls or complex logic."
  resolution: "Increase Lambda timeout (max 900s), optimize API calls, or reduce evaluation scope."

- symptoms: "Lambda throttled — TooManyRequestsException"
  diagnosis: "Lambda concurrency limit reached due to many resources triggering evaluation simultaneously."
  resolution: "Increase reserved concurrency or request a Lambda concurrency limit increase."

- symptoms: "PutEvaluations call fails with AccessDenied"
  diagnosis: "The Lambda execution role lacks config:PutEvaluations permission."
  resolution: "Add config:PutEvaluations to the Lambda execution role policy."

- symptoms: "Rule shows error but Lambda logs show successful execution"
  diagnosis: "The Lambda function returns evaluations in an incorrect format."
  resolution: "Ensure the response includes ComplianceResourceType, ComplianceResourceId, ComplianceType, and OrderingTimestamp."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix Lambda function code | YELLOW | Code change; previous version restorable |
| Fix Lambda execution role | YELLOW | Permission change; reversible |
| Increase Lambda timeout/memory | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- Lambda function changes affecting compliance evaluation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-config-rule-evaluation-status` | LOW | Evaluation status |
| `get-function` | MEDIUM | Lambda configuration |
| Lambda CloudWatch logs | MEDIUM | Evaluation details |
| Lambda error/throttle metrics | LOW | Operational metrics |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Lambda code fix | Redeploy previous Lambda version |
| IAM role fix | Revert IAM policy to previous version |
| Timeout/memory change | Revert Lambda configuration |

## Output Format

```yaml
root_cause: "custom_rule_lambda_error — <specific_cause>"
evidence:
  - type: lambda_function
    content: "<get-function output>"
  - type: lambda_errors
    content: "<CloudWatch metrics for errors/throttles>"
  - type: lambda_logs
    content: "<relevant log entries>"
severity: HIGH
mitigation:
  immediate: "Fix the Lambda function issue and force rule re-evaluation"
  long_term: "Add monitoring, error handling, and appropriate concurrency settings"
```

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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
