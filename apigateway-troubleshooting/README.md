# API Gateway Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon API Gateway problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for API Gateway (REST API, HTTP API, WebSocket API) when the console alone isn't enough — integration debugging, authorization failures, throttling analysis, deployment issues, custom domain configuration, WebSocket troubleshooting, mapping template errors, request validation, binary media types, and performance optimization.

### Activate When

- Lambda proxy integration errors (malformed response)
- Lambda custom integration mapping failures
- HTTP integration timeout (29-second limit)
- VPC link connectivity issues
- 4xx client errors (400, 403, 404, 429)
- 5xx server errors (500, 502, 503, 504)
- CORS preflight or response header failures
- Binary media type encoding/decoding issues
- Lambda authorizer failures or caching problems
- Cognito user pool authorizer issues
- IAM authorization denied
- Account-level throttling (429 Too Many Requests)
- Stage or method-level throttling
- Usage plan quota exceeded
- Deployment failures or missing changes
- Stage configuration issues (variables, logging, caching)
- Canary deployment problems
- Custom domain name setup failures
- ACM certificate issues for custom domains
- WebSocket connection failures ($connect route)
- WebSocket route selection expression issues
- WebSocket message delivery failures
- Execution log configuration issues
- Access log format or delivery problems
- Mapping template (VTL) errors
- Request validation failures

---

## Skill Structure

```
apigateway-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-lambda-proxy-errors.md
    ├── A2-lambda-custom-integration.md
    ├── A3-http-integration-timeout.md
    ├── A4-vpc-link-issues.md
    ├── B1-4xx-client-errors.md
    ├── B2-5xx-server-errors.md
    ├── B3-cors-failures.md
    ├── B4-binary-media-types.md
    ├── C1-lambda-authorizer-failures.md
    ├── C2-cognito-authorizer.md
    ├── C3-iam-authorization.md
    ├── D1-account-throttling.md
    ├── D2-stage-method-throttling.md
    ├── D3-usage-plan-limits.md
    ├── E1-deployment-failures.md
    ├── E2-stage-configuration.md
    ├── E3-canary-deployments.md
    ├── F1-domain-name-setup.md
    ├── F2-certificate-issues.md
    ├── G1-websocket-connection-failures.md
    ├── G2-websocket-route-selection.md
    ├── G3-websocket-message-delivery.md
    ├── H1-execution-logging.md
    ├── H2-access-logging.md
    ├── I1-mapping-templates.md
    ├── I2-request-validation.md
    ├── Z1-general-troubleshooting.md
    ├── apigateway-guardrails.md
    └── apigateway-hallucination-patterns.yaml
```

---

## Runbook Library (32 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Integration** | A1–A4 | Lambda proxy errors, Lambda custom integration, HTTP integration timeout, VPC link issues |
| **B — Errors** | B1–B4 | 4xx client errors, 5xx server errors, CORS failures, binary media type issues |
| **C — Authorization** | C1–C3 | Lambda authorizer failures, Cognito authorizer, IAM authorization |
| **D — Throttling** | D1–D3 | Account throttling, stage/method throttling, usage plan limits |
| **E — Deployment** | E1–E3 | Deployment failures, stage configuration, canary deployments |
| **F — Custom Domains** | F1–F2 | Domain name setup, certificate issues |
| **G — WebSocket** | G1–G3 | Connection failures, route selection, message delivery |
| **H — Logging** | H1–H2 | Execution logging, access logging |
| **I — Request/Response** | I1–I2 | Mapping templates, request validation |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/apigateway-guardrails.md` covering 29-second timeout hard limit, deployment required for REST API, CORS OPTIONS method requirement, API keys not for authentication, authorizer caching behavior, throttling hierarchy, Lambda proxy response format, stage variables scope, binary media type configuration, custom domain ACM region requirements, WebSocket route selection, and VPC link requirements.

---

## Investigation Workflow

1. **Triage** — List APIs, check stages/deployments, review CloudWatch error rates and latency metrics
2. **Deep Dive** — Examine resources, methods, integrations, authorizers, gateway responses, request validators
3. **Detailed** — Inspect execution logs, X-Ray traces, CloudTrail events, usage plan consumption

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `apigateway:*`, `lambda:GetFunction`, `lambda:InvokeFunction`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `logs:FilterLogEvents`, `xray:GetTraceSummaries`, `acm:DescribeCertificate`, `wafv2:GetWebACL`, `cognito-idp:DescribeUserPool`
- API Gateway execution logs enabled (recommended)
- X-Ray tracing enabled (recommended)

---

## Usage Examples

```
# List all REST APIs
aws apigateway get-rest-apis

# Get API resources and methods
aws apigateway get-resources --rest-api-id abc123def4

# Test invoke a method
aws apigateway test-invoke-method --rest-api-id abc123def4 \
  --resource-id res123 --http-method GET

# Check error rates
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway \
  --metric-name 5XXError --dimensions Name=ApiName,Value=MyAPI \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z \
  --period 300 --statistics Sum

# Check throttling
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway \
  --metric-name Count --dimensions Name=ApiName,Value=MyAPI \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z \
  --period 60 --statistics Sum
```

---

## License

MIT-0
