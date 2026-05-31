---
title: "A1 — Lambda Proxy Integration Errors"
description: "Diagnose Lambda proxy integration failures including malformed responses and 502 errors"
status: active
severity: HIGH
triggers:
  - "Lambda proxy"
  - "502"
  - "malformed Lambda proxy response"
  - "internal server error"
  - "proxy integration"
owner: devops-agent
objective: "Resolve Lambda proxy integration errors by validating response format, permissions, and function execution"
context: "Lambda proxy integration passes the entire API Gateway request to Lambda and expects a specific response format: { statusCode, headers, body }. The body must be a string. Missing or malformed fields cause API Gateway to return 502 Internal Server Error with 'Malformed Lambda proxy response' in execution logs."
---

## Phase 1 — Triage

MUST:
- Check execution logs for malformed response: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "Malformed" --start-time <epoch-ms>`
- Test invoke the method: `aws apigateway test-invoke-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method>`
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Verify integration type is AWS_PROXY: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'type'`
- Check Lambda function permissions: `aws lambda get-policy --function-name <function-name>`

SHOULD:
- Verify Lambda function exists and is not in error state: `aws lambda get-function --function-name <function-name>`
- Check if Lambda is hitting timeout: `aws lambda get-function-configuration --function-name <function-name> --query 'Timeout'`
- Check Lambda concurrent execution limits: `aws lambda get-function-concurrency --function-name <function-name>`

MAY:
- Enable X-Ray tracing for detailed latency breakdown
- Check Lambda CloudWatch metrics for errors and throttles

## Phase 2 — Remediate

MUST:
- Ensure Lambda returns the correct format:
  ```json
  {
    "statusCode": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": "{\"message\": \"success\"}"
  }
  ```
- The body field MUST be a string — use JSON.stringify() for objects
- Ensure statusCode is present and is a number
- Add error handling in Lambda to always return valid proxy response format

SHOULD:
- Add try/catch wrapper to ensure error responses also follow the format
- Set Lambda timeout to less than 29 seconds (API Gateway hard limit)
- Add CORS headers (Access-Control-Allow-Origin) in the response if needed

MAY:
- Use Lambda Powertools for structured response formatting
- Implement response validation in Lambda before returning

## Common Issues

- symptoms: "502 Internal Server Error with 'Malformed Lambda proxy response'"
  diagnosis: "Lambda function returns a raw object without statusCode or returns body as object instead of string."
  resolution: "Return { statusCode: 200, headers: {...}, body: JSON.stringify(data) }."

- symptoms: "502 after Lambda timeout"
  diagnosis: "Lambda execution exceeds 29-second API Gateway timeout."
  resolution: "Reduce Lambda timeout below 29s or use async invocation pattern."

- symptoms: "502 with 'Execution failed due to configuration error: Invalid permissions'"
  diagnosis: "API Gateway does not have permission to invoke the Lambda function."
  resolution: "Add Lambda resource-based policy: aws lambda add-permission --function-name <fn> --statement-id apigateway --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn 'arn:aws:execute-api:<region>:<account>:<api-id>/*'"


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If throttling settings were modified, restore original rate/burst limits: `aws apigateway update-stage --rest-api-id <id> --stage-name <stage> --patch-operations <revert>`
3. If integration settings were changed, restore original integration configuration and redeploy
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "lambda_proxy — <specific_cause>"
evidence:
  - type: execution_logs
    content: "<API Gateway execution log entries>"
  - type: lambda_response
    content: "<actual Lambda response format>"
  - type: lambda_logs
    content: "<Lambda function error output>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda response format or permissions"
  long_term: "Implement response wrapper and structured error handling"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"
