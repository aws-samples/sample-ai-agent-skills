---
title: "G1 — WebSocket Connection Failures"
description: "Diagnose WebSocket API connection failures including $connect route and authorization"
status: active
severity: HIGH
triggers:
  - "WebSocket"
  - "$connect"
  - "connection failed"
  - "WebSocket handshake"
  - "upgrade"
  - "connection refused"
owner: devops-agent
objective: "Resolve WebSocket API connection failures by analyzing $connect route, authorization, and integration configuration"
context: "WebSocket API connections start with an HTTP upgrade request that invokes the $connect route. Authorization happens at $connect time — if the $connect integration returns an error or the authorizer denies, the connection is rejected. The $connect route integration must return a 200 status for the connection to succeed. Common failures include missing $connect route, authorizer errors, and Lambda integration failures."
---

## Phase 1 — Triage

MUST:
- Check WebSocket API configuration: `aws apigatewayv2 get-api --api-id <api-id>`
- Check $connect route: `aws apigatewayv2 get-routes --api-id <api-id> --query "Items[?RouteKey=='$connect']"`
- Check $connect integration: `aws apigatewayv2 get-integrations --api-id <api-id>`
- Check $connect authorizer if configured: `aws apigatewayv2 get-authorizers --api-id <api-id>`
- Check execution logs: `aws logs filter-log-events --log-group-name <log-group> --filter-pattern "connect" --start-time <epoch-ms>`
- Test WebSocket connection: `wscat -c "wss://<api-id>.execute-api.<region>.amazonaws.com/<stage>"`

SHOULD:
- Check Lambda function logs for $connect handler: `aws logs filter-log-events --log-group-name /aws/lambda/<connect-function> --start-time <epoch-ms>`
- Verify stage is deployed: `aws apigatewayv2 get-stages --api-id <api-id>`
- Check Lambda permissions for WebSocket API invocation

MAY:
- Check CloudTrail for WebSocket API configuration changes
- Verify client WebSocket library compatibility

## Phase 2 — Remediate

MUST:
- Ensure $connect route exists and has a valid integration
- Ensure $connect Lambda returns statusCode 200 for successful connections
- Fix authorizer configuration if authorization is failing
- Verify Lambda has permission to be invoked by the WebSocket API

SHOULD:
- Store connection ID in DynamoDB on $connect for later message delivery
- Implement proper error handling in $connect Lambda
- Set up $disconnect route to clean up connection records

MAY:
- Implement connection authentication via query string parameters or headers
- Add connection rate limiting

## Common Issues

- symptoms: "WebSocket connection immediately closes"
  diagnosis: "$connect Lambda returns non-200 status or throws an error."
  resolution: "Ensure $connect Lambda returns { statusCode: 200 }. Check Lambda logs for errors."

- symptoms: "403 on WebSocket connection attempt"
  diagnosis: "Authorizer on $connect route denies the connection."
  resolution: "Check authorizer configuration and token/credentials. Test authorizer independently."

- symptoms: "Connection hangs and times out"
  diagnosis: "$connect Lambda exceeds timeout or integration is misconfigured."
  resolution: "Check Lambda timeout and integration configuration. Ensure integration type is correct."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
5. If CORS or method configuration was changed, restore original settings and redeploy the API
6. If WebSocket route configuration was changed, restore original route integration and redeploy
7. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "websocket_connect — <specific_cause>"
evidence:
  - type: connect_route
    content: "<$connect route and integration configuration>"
  - type: authorizer
    content: "<authorizer configuration if applicable>"
  - type: lambda_logs
    content: "<$connect Lambda execution logs>"
severity: HIGH
mitigation:
  immediate: "Fix $connect route integration, authorizer, or Lambda function"
  long_term: "Implement robust connection handling with DynamoDB tracking"
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
