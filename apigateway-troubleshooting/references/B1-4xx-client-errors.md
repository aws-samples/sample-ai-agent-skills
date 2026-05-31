---
title: "B1 — 4xx Client Errors"
description: "Diagnose API Gateway 4xx client errors including 400, 403, 404, and 429"
status: active
severity: HIGH
triggers:
  - "4xx"
  - "400"
  - "403"
  - "404"
  - "429"
  - "bad request"
  - "forbidden"
  - "not found"
  - "missing authentication token"
owner: devops-agent
objective: "Identify and resolve 4xx client errors by analyzing request validation, authorization, resource paths, and throttling"
context: "4xx errors indicate client-side issues. 400 = bad request (validation failure), 403 = forbidden (auth failure, WAF block, resource policy deny), 404 = not found (wrong path, missing deployment), 429 = throttled. API Gateway returns specific error messages in execution logs that identify the exact cause."
---

## Phase 1 — Triage

MUST:
- Check 4xx error rate: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 4XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check execution logs for error details: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --start-time <epoch-ms>`
- Verify resource exists: `aws apigateway get-resources --rest-api-id <api-id> --query 'items[*].{id:id,path:path,methods:resourceMethods}'`
- Check authorization configuration: `aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query '{authType:authorizationType,apiKeyRequired:apiKeyRequired}'`
- Check if deployment is current: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'deploymentId'`

SHOULD:
- Check gateway responses for custom error messages: `aws apigateway get-gateway-responses --rest-api-id <api-id>`
- Check WAF if associated: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'webAclArn'`
- Check resource policy: `aws apigateway get-rest-api --rest-api-id <api-id> --query 'policy'`

MAY:
- Check request validators: `aws apigateway get-request-validators --rest-api-id <api-id>`
- Review CloudTrail for recent API changes

## Phase 2 — Remediate

MUST:
- For 403 "Missing Authentication Token": verify the resource path exists and is deployed. This error often means the path is wrong, not an auth issue
- For 403 with authorization: check authorizer configuration, token format, and IAM policies
- For 404: verify resource path, HTTP method, and that a deployment exists for the stage
- For 429: check throttling at account, stage, method, and usage plan levels

SHOULD:
- Customize gateway responses for better error messages
- Add request validation to return clear 400 errors instead of backend failures
- Review and update resource policies if using IP-based or VPC-based restrictions

MAY:
- Implement structured error responses across all methods
- Add CloudWatch alarms for 4xx error rate spikes

## Common Issues

- symptoms: "403 Missing Authentication Token on valid path"
  diagnosis: "The resource path does not exist or is not deployed. API Gateway returns this misleading error."
  resolution: "Verify the exact resource path and ensure a deployment exists for the stage."

- symptoms: "403 Forbidden with valid credentials"
  diagnosis: "Resource policy, WAF rule, or IP restriction is blocking the request."
  resolution: "Check resource policy, WAF web ACL, and stage-level access restrictions."

- symptoms: "429 Too Many Requests"
  diagnosis: "Request rate exceeds throttling limits at account, stage, or usage plan level."
  resolution: "Check throttling at all levels. Increase limits or implement client-side retry with backoff."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
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
5. If CORS or method configuration was changed, restore original settings and redeploy the API
6. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "4xx_error — <specific_cause>"
evidence:
  - type: error_code
    content: "<specific 4xx status and message>"
  - type: execution_logs
    content: "<API Gateway execution log details>"
  - type: configuration
    content: "<relevant auth/resource/throttle config>"
severity: HIGH
mitigation:
  immediate: "Fix resource path, authorization, or throttling configuration"
  long_term: "Implement request validation, custom error responses, and monitoring"
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
