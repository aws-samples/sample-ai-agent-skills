---
title: "B1 — Lambda Invocation Errors"
description: "Diagnose Lambda function invocation failures in Object Lambda"
status: active
severity: HIGH
triggers: ["Lambda invocation failed", "500 error from OLAP", "function not invoked", "InvocationFailed"]
owner: devops-agent
objective: "Resolve Lambda invocation errors when accessing objects through OLAPs"
context: "Object Lambda invokes the configured Lambda function for each request. Invocation failures return 500 errors to the caller."
---

## Phase 1 — Triage

MUST:
- Check Lambda function exists and is active: `aws lambda get-function --function-name <function-name>`
- Check Lambda invocation errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Review Lambda logs for errors: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --filter-pattern "ERROR"`

SHOULD:
- Check Lambda concurrent execution limits: `aws lambda get-account-settings`
- Verify Lambda execution role has required permissions
- Check if the function is throttled: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Throttles --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Test Lambda function directly: `aws lambda invoke --function-name <function-name> --payload file://test-event.json output.json`
- Check if VPC configuration is blocking Lambda network access

## Phase 2 — Remediate

MUST:
- Fix Lambda function code errors identified in logs
- Ensure Lambda execution role has s3-object-lambda:WriteGetObjectResponse permission
- Resolve throttling by requesting concurrency limit increase or configuring reserved concurrency

SHOULD:
- Add error handling and logging to the Lambda function
- Configure dead letter queue for failed invocations
- Set appropriate memory and timeout for the function

MAY:
- Enable X-Ray tracing for detailed invocation analysis
- Implement retry logic in the Lambda function for transient S3 errors

## Common Issues

- symptoms: "Lambda function invoked but returns 500 to caller"
  diagnosis: "Function completes without calling WriteGetObjectResponse."
  resolution: "Ensure the function calls s3.writeGetObjectResponse() with requestRoute and requestToken from the event."

- symptoms: "Lambda function not invoked at all — immediate 500 error"
  diagnosis: "Lambda function does not exist, is in wrong region, or IAM permissions prevent invocation."
  resolution: "Verify function ARN in OLAP config. Check s3-object-lambda service has lambda:InvokeFunction permission."

## Safety Ratings

```
safety_ratings:
  - "lambda get-function: GREEN — read-only function metadata"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log search"
  - "lambda invoke: YELLOW — executes function code, may have side effects"
```

## Escalation Conditions

- Lambda function consistently fails with unhandled exceptions after code fixes
- Throttling persists after concurrency limit increase
- Function timeout at 60 seconds cannot be optimized further
- Cross-account Lambda invocation requires organizational policy changes

## Data Sensitivity

- `aws lambda get-function`: MEDIUM — may reveal environment variables and VPC config
- `aws logs filter-log-events`: HIGH — may contain request data, object content, or PII in logs
- `aws lambda invoke`: MEDIUM — executes function with test payload
- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metrics only

## Prohibited Actions

- NEVER invoke a production Lambda function with arbitrary test payloads without approval
- NEVER modify Lambda function code directly in production — use deployment pipelines
- NEVER remove the Lambda execution role's WriteGetObjectResponse permission
- NEVER set Lambda concurrency to 0 (this disables the function entirely)

## Phase 3 — Rollback

1. If Lambda function code was updated and broke, roll back to previous version: `aws lambda update-function-code --function-name <name> --s3-bucket <bucket> --s3-key <previous-version-key>`
2. If concurrency settings were changed, restore previous value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <previous-value>`
3. If execution role was modified, restore previous policy
4. Verify OLAP requests succeed after rollback
5. Document the incident and update the Lambda function's test suite

## Output Format

```yaml
root_cause: "lambda_invocation — <specific_cause>"
evidence:
  - type: invocation_error
    content: "<error details from logs>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda function or permissions"
  long_term: "Add comprehensive error handling and monitoring"
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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"
