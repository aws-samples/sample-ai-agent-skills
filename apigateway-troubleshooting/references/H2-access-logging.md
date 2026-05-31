---
title: "H2 — Access Logging Issues"
description: "Diagnose API Gateway access logging configuration and log format problems"
status: active
severity: MEDIUM
triggers:
  - "access log"
  - "access logging"
  - "log format"
  - "CLF"
  - "JSON log"
  - "custom log format"
owner: devops-agent
objective: "Resolve access logging issues by configuring log destinations, formats, and permissions"
context: "API Gateway access logs provide customizable request-level logging with fields like requestId, IP, method, path, status, latency. Access logs can be sent to CloudWatch Logs, Firehose, or (for HTTP API) CloudWatch Logs only. The log format uses $context variables. Access logging is separate from execution logging and requires its own configuration per stage."
---

## Phase 1 — Triage

MUST:
- Check access log settings: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'accessLogSettings'`
- Verify destination log group/ARN exists
- Check log format for valid $context variables
- For HTTP API: `aws apigatewayv2 get-stage --api-id <api-id> --stage-name '$default' --query 'AccessLogSettings'`
- Check if logs are appearing in the destination: `aws logs filter-log-events --log-group-name <access-log-group> --start-time <epoch-ms> --limit 10`

SHOULD:
- Verify CloudWatch Logs role has permissions for the destination
- Check log group retention policy and storage costs
- Validate log format syntax

MAY:
- Check Firehose delivery stream status if using Firehose destination
- Review log format for missing useful fields

## Phase 2 — Remediate

MUST:
- Configure access logging with log group ARN and format:
  ```
  aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> \
    --patch-operations op=replace,path=/accessLogSettings/destinationArn,value=arn:aws:logs:<region>:<account>:log-group:<group> \
    op=replace,path=/accessLogSettings/format,value='{"requestId":"$context.requestId","ip":"$context.identity.sourceIp","method":"$context.httpMethod","path":"$context.resourcePath","status":"$context.status","latency":"$context.responseLatency"}'
  ```
- Ensure the log group exists before configuring access logging
- Verify IAM permissions for writing to the log destination

SHOULD:
- Include essential fields: requestId, sourceIp, httpMethod, resourcePath, status, responseLatency, integrationLatency
- Use JSON format for easier parsing and analysis
- Set appropriate log retention period

MAY:
- Add custom fields: $context.authorizer.claims.sub, $context.identity.userAgent
- Set up CloudWatch Insights queries for access log analysis
- Stream to Firehose for S3/Elasticsearch delivery

## Common Issues

- symptoms: "Access logs not appearing despite configuration"
  diagnosis: "Destination log group does not exist or IAM role lacks write permissions."
  resolution: "Create the log group first, then configure access logging. Verify IAM permissions."

- symptoms: "Access log entries have empty fields"
  diagnosis: "Log format references $context variables that are not available for the request type."
  resolution: "Use only supported $context variables. Some variables are only available for specific auth types."

- symptoms: "Access logging works for REST API but not HTTP API"
  diagnosis: "HTTP API uses different stage configuration and $context variable names."
  resolution: "Use apigatewayv2 commands for HTTP API. Check HTTP API-specific $context variables."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
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
3. If integration settings were changed, restore original integration configuration and redeploy
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "access_logging — <specific_cause>"
evidence:
  - type: access_log_settings
    content: "<destination ARN and log format>"
  - type: log_destination
    content: "<log group/Firehose status>"
  - type: sample_logs
    content: "<recent access log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix log destination, format, or permissions"
  long_term: "Implement comprehensive access logging with analysis pipeline"
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
