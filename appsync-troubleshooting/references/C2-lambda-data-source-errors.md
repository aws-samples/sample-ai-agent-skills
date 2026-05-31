---
title: "C2 — Lambda Data Source Errors"
description: "Diagnose AppSync Lambda data source invocation and response errors"
status: active
severity: HIGH
triggers:
  - "Lambda error"
  - "Lambda data source"
  - "Lambda invocation failed"
  - "Lambda timeout"
  - "function error"
owner: devops-agent
objective: "Identify and resolve Lambda data source errors in AppSync resolvers"
context: "Lambda data sources invoke Lambda functions for resolver logic. Common issues include IAM permission errors, Lambda cold starts causing timeouts, incorrect request/response payload format, Lambda function errors, and misconfigured batch invocations."
---

## Phase 1 — Triage

MUST:
- Get the data source configuration: `aws appsync get-data-source --api-id <api-id> --name <ds-name>`
- Check the Lambda function exists and is accessible: `aws lambda get-function --function-name <function-name>`
- Check the IAM role permissions for lambda:InvokeFunction
- Check Lambda function errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check Lambda function duration: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`

SHOULD:
- Check Lambda CloudWatch Logs for function errors: `/aws/lambda/<function-name>`
- Verify the Lambda function returns the expected response format
- Check if the Lambda function timeout + cold start exceeds the 30s resolver timeout

MAY:
- Check if batch invocation (BatchInvoke) is configured correctly
- Verify Lambda concurrency limits aren't being hit

## Phase 2 — Remediate

MUST:
- Ensure the IAM role has lambda:InvokeFunction permission for the specific function ARN
- Ensure the role trust policy allows appsync.amazonaws.com
- Fix Lambda function errors (check function logs)

SHOULD:
- Set Lambda timeout to less than 30 seconds (AppSync resolver limit)
- Use provisioned concurrency for latency-sensitive resolvers
- Return properly formatted responses from the Lambda function

MAY:
- Use direct Lambda resolvers (no mapping template) for simpler configuration
- Implement batch invocation for list resolvers to avoid N+1

## Common Issues

- symptoms: "Lambda invocation failed with access denied"
  diagnosis: "IAM role lacks lambda:InvokeFunction permission."
  resolution: "Add lambda:InvokeFunction permission for the function ARN to the data source role."

- symptoms: "Lambda resolver intermittently returns null"
  diagnosis: "Lambda function throws an unhandled error. AppSync returns null for unhandled errors."
  resolution: "Add error handling in the Lambda function. Check Lambda CloudWatch Logs."

- symptoms: "Resolver times out but Lambda function completes in 5 seconds"
  diagnosis: "Lambda cold start adds latency pushing total time over 30 seconds."
  resolution: "Use provisioned concurrency or optimize Lambda package size."

## Output Format

```yaml
root_cause: "lambda_data_source_error — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<Lambda function ARN and IAM role>"
  - type: lambda_metrics
    content: "<Duration and Error metrics>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or Lambda function errors"
  long_term: "Add provisioned concurrency and proper error handling"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure IAM role has lambda:InvokeFunction | YELLOW | IAM policy change |
| Ensure role trust policy correct | YELLOW | IAM trust policy change |
| Fix Lambda function errors | YELLOW | Lambda code change |
| Set Lambda timeout < 30s | YELLOW | Lambda configuration change |
| Use provisioned concurrency | YELLOW | Lambda scaling change |
| Use direct Lambda resolvers | YELLOW | Resolver configuration change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Lambda function ARN**: Function identity — INTERNAL
- **Lambda metrics**: Duration and error rates — INTERNAL
- **Lambda logs**: Function execution output — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER grant lambda:* wildcard permissions to data source roles
- NEVER deploy Lambda function changes without testing

## Phase 3 — Rollback

1. Revert Lambda function to previous version: `aws lambda update-function-code --function-name <name> --s3-bucket <bucket> --s3-key <previous-key>`
2. Revert IAM role policy changes
3. Revert provisioned concurrency settings if changed
4. Verify resolver returns expected results after rollback
5. Monitor Lambda error metrics for 15 minutes

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
