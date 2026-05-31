---
title: "B2 — 5xx Server Errors"
description: "Diagnose API Gateway 5xx server errors including 500, 502, 503, and 504"
status: active
severity: CRITICAL
triggers:
  - "5xx"
  - "500"
  - "502"
  - "503"
  - "504"
  - "internal server error"
  - "bad gateway"
  - "service unavailable"
owner: devops-agent
objective: "Identify and resolve 5xx server errors by analyzing integration failures, backend health, and API Gateway configuration"
context: "5xx errors indicate server-side issues. 500 = internal error (mapping template failure, authorizer error), 502 = bad gateway (malformed Lambda response, integration failure), 503 = service unavailable (throttling at service level), 504 = gateway timeout (backend exceeds 29s limit). Execution logs contain the specific failure reason."
---

## Phase 1 — Triage

MUST:
- Check 5xx error rate and pattern: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 5XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check execution logs: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "Execution failed" --start-time <epoch-ms>`
- Check integration latency for timeouts: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name IntegrationLatency --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics p99`
- Test invoke the failing method: `aws apigateway test-invoke-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method>`
- Check Lambda errors if Lambda integration: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check X-Ray traces for error details: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'fault = true'`
- Check Lambda throttling: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Throttles --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check backend health for HTTP integrations

MAY:
- Check CloudTrail for recent configuration changes
- Review API Gateway service health dashboard

## Phase 2 — Remediate

MUST:
- For 502: fix Lambda response format (statusCode, headers, body as string) or fix integration configuration
- For 504: optimize backend to respond within 29 seconds or implement async pattern
- For 500: fix mapping templates, authorizer errors, or internal configuration issues
- For 503: check if API Gateway service is experiencing issues or if account limits are hit

SHOULD:
- Add structured error handling in Lambda functions
- Implement retry logic with exponential backoff in clients
- Set up CloudWatch alarms for 5xx error rate thresholds

MAY:
- Implement circuit breaker pattern for unreliable backends
- Add canary deployments to catch issues before full rollout

## Common Issues

- symptoms: "502 Bad Gateway with Lambda integration"
  diagnosis: "Lambda returns malformed proxy response or Lambda function errors out."
  resolution: "Check Lambda response format and function logs. Ensure statusCode and string body."

- symptoms: "504 Gateway Timeout consistently"
  diagnosis: "Backend processing exceeds 29-second hard limit."
  resolution: "Optimize backend or implement async pattern with Step Functions or SQS."

- symptoms: "Intermittent 500 errors"
  diagnosis: "Mapping template errors, authorizer failures, or transient backend issues."
  resolution: "Check execution logs for specific error. Fix templates or authorizer configuration."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
2. If authorizer configuration was changed, restore the original authorizer settings and redeploy the stage
3. If throttling settings were modified, restore original rate/burst limits: `aws apigateway update-stage --rest-api-id <id> --stage-name <stage> --patch-operations <revert>`
4. If integration settings were changed, restore original integration configuration and redeploy
5. If custom domain or base path mapping was changed, restore original mapping configuration
6. If CORS or method configuration was changed, restore original settings and redeploy the API
7. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "5xx_error — <specific_cause>"
evidence:
  - type: error_pattern
    content: "<5xx error rate and timing>"
  - type: execution_logs
    content: "<specific error messages from execution logs>"
  - type: integration_health
    content: "<backend/Lambda health status>"
severity: CRITICAL
mitigation:
  immediate: "Fix integration response format, backend timeout, or configuration error"
  long_term: "Implement error handling, monitoring, and async patterns"
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
