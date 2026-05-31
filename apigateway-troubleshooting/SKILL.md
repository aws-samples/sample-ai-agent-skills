---
name: apigateway-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon API Gateway problems
  including REST API, HTTP API, and WebSocket API types by analyzing
  configurations, integrations, authorizers, deployments, and following
  structured runbooks. Activate when: Lambda proxy errors, integration
  timeouts, 4xx/5xx errors, CORS failures, authorization issues (Lambda
  authorizer, Cognito, IAM), throttling, deployment failures, custom domain
  problems, WebSocket connection issues, mapping template errors, request
  validation failures, binary media type issues, stage variable problems,
  canary deployment issues, or the user says something is wrong with API
  Gateway without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with API Gateway, Lambda, CloudWatch,
  CloudTrail, X-Ray, ACM, WAF, IAM, Cognito, and optionally VPC and
  Route 53 permissions.
---

# API Gateway Diagnostics

## When to use

Any API Gateway investigation where the console alone is insufficient — integration debugging, authorization failures, throttling analysis, deployment issues, custom domain configuration, WebSocket troubleshooting, mapping template errors, or performance optimization.

## Investigation workflow

### Step 1 — Collect and triage

```
aws apigateway get-rest-apis --query 'items[*].{id:id,name:name,endpointConfiguration:endpointConfiguration}'
version: "1.0.0"
last_updated: "2025-04-12"
aws apigateway get-stages --rest-api-id <api-id>
aws apigateway get-deployments --rest-api-id <api-id>
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 4XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 5XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name Latency --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name Count --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws apigateway get-resources --rest-api-id <api-id>
aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method>
aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method>
aws apigateway test-invoke-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method>
aws apigateway get-authorizers --rest-api-id <api-id>
aws apigateway get-gateway-responses --rest-api-id <api-id>
aws apigateway get-models --rest-api-id <api-id>
aws apigateway get-request-validators --rest-api-id <api-id>
```

### Step 3 — Detailed investigation

```
aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --start-time <epoch-ms> --end-time <epoch-ms>
aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'service("<api-name>")'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=apigateway.amazonaws.com --max-results 20
aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query '{logging:methodSettings,variables:variables,cacheCluster:cacheClusterEnabled}'
aws apigateway get-usage --usage-plan-id <plan-id> --key-id <key-id> --start-date <YYYY-MM-DD> --end-date <YYYY-MM-DD>
```

Read `references/apigateway-guardrails.md` before concluding on any API Gateway issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-rest-apis` | List all REST APIs |
| `get-resources` | List resources and methods |
| `get-method` | Method configuration (auth, request params) |
| `get-integration` | Integration type, URI, mappings |
| `test-invoke-method` | Test method invocation from API Gateway |
| `get-authorizers` | List authorizers and configuration |
| `get-stages` | Stage configurations and settings |
| `get-deployments` | Deployment history |
| `get-gateway-responses` | Custom gateway responses |
| `get-usage` | Usage plan consumption |
| `get-domain-names` | Custom domain name configuration |
| `get-base-path-mappings` | Base path to API/stage mappings |

## Gotchas: API Gateway

- 29-second integration timeout: REST API has a hard limit of 29 seconds for integration timeout. This CANNOT be increased. HTTP API has a 30-second timeout. If your backend needs more time, use asynchronous patterns (Step Functions, SQS).
- REST API vs HTTP API vs WebSocket API: REST API supports more features (request validation, WAF, resource policies, usage plans, API keys, caching). HTTP API is cheaper, faster, supports OIDC/OAuth2 natively, and has automatic deployments. WebSocket API is for bidirectional real-time communication.
- Lambda proxy vs custom integration: Lambda proxy passes the entire request to Lambda and expects a specific response format (statusCode, headers, body). Custom integration allows request/response mapping templates but requires more configuration.
- Stage variables: can be used in integration URIs, Lambda function names, and mapping templates. They are NOT environment variables — they are API Gateway configuration values resolved at request time.
- Deployment required after changes: REST API changes (methods, integrations, models, authorizers) are NOT live until you create a new deployment. HTTP API has automatic deployments by default.
- CORS requires OPTIONS method + headers: For REST API, CORS requires an OPTIONS method with mock integration returning Access-Control-Allow-Origin, Access-Control-Allow-Methods, and Access-Control-Allow-Headers. The actual method must also return these headers. HTTP API has built-in CORS configuration.
- Authorizer caching: Lambda authorizer results can be cached (TTL 0-3600 seconds, default 300s). The cache key is the authorization token or identity sources. TTL 0 disables caching. Cached results apply to ALL methods using that authorizer.
- Throttling hierarchy: Account-level default is 10,000 requests per second with 5,000 burst. Stage-level and method-level overrides can only REDUCE, not exceed account limits. Usage plans add per-client throttling via API keys.
- Binary media types: must be configured in API settings. For Lambda proxy, the request body is base64-encoded when Content-Type matches a binary media type. The response must set isBase64Encoded: true.
- Request/response mapping templates: use Apache Velocity Template Language (VTL). Available in REST API custom integrations only. Common variables: $input.body, $input.params(), $context, $stageVariables.
- API keys are NOT for authentication: API keys are for tracking and throttling via usage plans. They provide identification, not authorization. Use Lambda authorizers, Cognito, or IAM for authentication.
- Usage plans and throttling: usage plans associate API keys with throttle and quota limits. Quota resets at the beginning of each period (day/week/month). Throttle limits in usage plans are per-client.
- Custom domain names: edge-optimized APIs require ACM certificate in us-east-1. Regional APIs require ACM certificate in the same region as the API. Custom domains use base path mappings to route to APIs/stages.
- WebSocket $connect/$disconnect/$default routes: $connect is invoked on connection, $disconnect on disconnection, $default for unmatched route keys. Route selection expression (default: $request.body.action) determines which route handles a message.
- VPC link for private integrations: REST API uses VPC link v1 (NLB required). HTTP API uses VPC link v2 (ALB, NLB, or Cloud Map). VPC links must be in AVAILABLE status before use.
- WAF integration: only REST API supports WAF directly. HTTP API does not support WAF natively — use CloudFront in front of HTTP API for WAF protection.
- Request validation: REST API supports request validation for body, query strings, and headers using models (JSON Schema). HTTP API does not have built-in request validation — use Lambda or middleware.
- Gateway responses customization: REST API allows customizing error responses (ACCESS_DENIED, THROTTLED, etc.) with custom status codes, headers, and body templates. Useful for consistent error formats and CORS headers on errors.

## Anti-hallucination rules

1. Always cite specific API configurations, integration settings, or CloudWatch metrics as evidence.
2. The 29-second integration timeout for REST API is a HARD LIMIT. Never claim it can be increased.
3. REST API changes require deployment. Never claim changes are live immediately after API modification.
4. API keys are for identification and throttling, NOT authentication. Never recommend API keys as a security mechanism.
5. Lambda proxy integration requires a specific response format. Never claim any JSON response works.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 32 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Integration | A1-A4 | Lambda proxy errors, Lambda custom integration, HTTP integration timeout, VPC link issues |
| B — Errors | B1-B4 | 4xx client errors, 5xx server errors, CORS failures, binary media type issues |
| C — Authorization | C1-C3 | Lambda authorizer failures, Cognito authorizer, IAM authorization |
| D — Throttling | D1-D3 | Account throttling, stage/method throttling, usage plan limits |
| E — Deployment | E1-E3 | Deployment failures, stage configuration, canary deployments |
| F — Custom Domains | F1-F2 | Domain name setup, certificate issues |
| G — WebSocket | G1-G3 | Connection failures, route selection, message delivery |
| H — Logging | H1-H2 | Execution logging, access logging |
| I — Request/Response | I1-I2 | Mapping templates, request validation |
| Z — Catch-All | Z1 | General troubleshooting |
