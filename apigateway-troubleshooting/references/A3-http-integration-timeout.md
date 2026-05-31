---
title: "A3 — HTTP Integration Timeout"
description: "Diagnose HTTP/HTTP_PROXY integration timeouts and 504 errors"
status: active
severity: HIGH
triggers:
  - "timeout"
  - "504"
  - "integration timeout"
  - "29 seconds"
  - "gateway timeout"
  - "HTTP integration"
owner: devops-agent
objective: "Resolve HTTP integration timeout issues by identifying slow backends and implementing async patterns"
context: "REST API has a hard integration timeout limit of 29 seconds. HTTP API has a 30-second limit. When the backend does not respond within this window, API Gateway returns 504 Gateway Timeout. This limit cannot be increased. Common causes include slow database queries, external API calls, or large payload processing."
---

## Phase 1 — Triage

MUST:
- Check integration timeout setting: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'timeoutInMillis'`
- Check Latency and IntegrationLatency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name IntegrationLatency --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics p99`
- Check 5xx error count: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 5XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check execution logs for timeout entries: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "timed out" --start-time <epoch-ms>`
- Verify backend endpoint is reachable and responsive

SHOULD:
- Check X-Ray traces for latency breakdown: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'responsetime > 25'`
- Verify integration endpoint URL: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'uri'`
- Check if VPC link is involved and its status

MAY:
- Test backend directly to measure response time: `curl -w "time_total: %{time_total}\n" -o /dev/null -s <backend-url>`
- Check backend health and resource utilization

## Phase 2 — Remediate

MUST:
- Optimize backend to respond within 29 seconds (REST API) or 30 seconds (HTTP API)
- For long-running operations, implement async pattern:
  - Return 202 Accepted immediately with a task ID
  - Process asynchronously (Lambda async, Step Functions, SQS)
  - Provide a status polling endpoint

SHOULD:
- Set integration timeout to a value less than 29 seconds to fail fast
- Implement circuit breaker pattern for unreliable backends
- Add caching at the API Gateway stage level for cacheable responses

MAY:
- Use WebSocket API for long-running operations requiring real-time updates
- Implement request queuing with SQS for burst traffic

## Common Issues

- symptoms: "504 Gateway Timeout after exactly 29 seconds"
  diagnosis: "Backend processing exceeds the 29-second hard limit."
  resolution: "Optimize backend or implement async pattern. Cannot increase timeout beyond 29s."

- symptoms: "Intermittent 504 errors during peak traffic"
  diagnosis: "Backend becomes slow under load, exceeding timeout."
  resolution: "Scale backend, add caching, or implement request queuing."

- symptoms: "504 with VPC link integration"
  diagnosis: "Network latency through VPC link adds to total integration time."
  resolution: "Check VPC link health, NLB target health, and security group rules."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
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
2. If integration settings were changed, restore original integration configuration and redeploy
3. If CORS or method configuration was changed, restore original settings and redeploy the API
4. If WebSocket route configuration was changed, restore original route integration and redeploy
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "integration_timeout — <specific_cause>"
evidence:
  - type: integration_latency
    content: "<IntegrationLatency p99 metric>"
  - type: timeout_setting
    content: "<configured timeout in ms>"
  - type: execution_logs
    content: "<timeout log entries>"
severity: HIGH
mitigation:
  immediate: "Optimize backend response time or implement async pattern"
  long_term: "Implement caching, async processing, and circuit breaker patterns"
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
