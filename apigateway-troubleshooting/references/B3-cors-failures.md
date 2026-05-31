---
title: "B3 — CORS Failures"
description: "Diagnose Cross-Origin Resource Sharing (CORS) preflight and response header failures"
status: active
severity: HIGH
triggers:
  - "CORS"
  - "cross-origin"
  - "Access-Control-Allow-Origin"
  - "preflight"
  - "OPTIONS"
  - "blocked by CORS"
owner: devops-agent
objective: "Resolve CORS failures by configuring OPTIONS method, integration responses, and gateway responses correctly"
context: "CORS requires the server to return specific headers (Access-Control-Allow-Origin, Access-Control-Allow-Methods, Access-Control-Allow-Headers). For REST API, this requires an OPTIONS method with MOCK integration AND CORS headers on actual method responses AND gateway error responses. HTTP API has built-in CORS configuration. Missing CORS headers on ANY response (including errors) causes browser-side failures."
---

## Phase 1 — Triage

MUST:
- Check if OPTIONS method exists: `aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method OPTIONS`
- Check OPTIONS integration type (should be MOCK): `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method OPTIONS`
- Check OPTIONS integration response headers: `aws apigateway get-integration-response --rest-api-id <api-id> --resource-id <resource-id> --http-method OPTIONS --status-code 200`
- Check actual method integration response for CORS headers: `aws apigateway get-integration-response --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --status-code 200`
- Check gateway responses for CORS headers: `aws apigateway get-gateway-responses --rest-api-id <api-id>`
- For HTTP API: `aws apigatewayv2 get-api --api-id <api-id> --query 'CorsConfiguration'`

SHOULD:
- Test preflight request: `curl -X OPTIONS -H "Origin: https://example.com" -H "Access-Control-Request-Method: POST" -v https://<api-url>/path`
- Test actual request headers: `curl -H "Origin: https://example.com" -v https://<api-url>/path`
- Check if Lambda proxy response includes CORS headers

MAY:
- Check browser developer console for specific CORS error message
- Verify allowed origins, methods, and headers match the client request

## Phase 2 — Remediate

MUST:
- For REST API, ensure OPTIONS method returns:
  - Access-Control-Allow-Origin: <origin or *>
  - Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
  - Access-Control-Allow-Headers: Content-Type,Authorization,X-Api-Key
- Ensure actual method responses include Access-Control-Allow-Origin header
- For Lambda proxy: add CORS headers in Lambda response
- Add CORS headers to gateway responses (DEFAULT_4XX, DEFAULT_5XX)

SHOULD:
- Use specific origins instead of * for production
- Include Access-Control-Allow-Credentials: true if sending cookies
- For HTTP API, configure CORS in the API settings: `aws apigatewayv2 update-api --api-id <id> --cors-configuration AllowOrigins="https://example.com",AllowMethods="GET,POST",AllowHeaders="Content-Type,Authorization"`

MAY:
- Add Access-Control-Max-Age to cache preflight responses
- Implement per-stage CORS configuration using stage variables

## Common Issues

- symptoms: "CORS error on preflight but OPTIONS returns 403"
  diagnosis: "OPTIONS method requires authentication or does not exist."
  resolution: "Set OPTIONS method authorization to NONE and ensure MOCK integration is configured."

- symptoms: "CORS works for 200 but fails on 4xx/5xx errors"
  diagnosis: "Gateway responses (DEFAULT_4XX, DEFAULT_5XX) do not include CORS headers."
  resolution: "Add Access-Control-Allow-Origin header to gateway responses."

- symptoms: "CORS works in test but fails with credentials"
  diagnosis: "Access-Control-Allow-Origin is set to * which is incompatible with credentials."
  resolution: "Set specific origin instead of * and add Access-Control-Allow-Credentials: true."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
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
root_cause: "cors — <specific_cause>"
evidence:
  - type: options_method
    content: "<OPTIONS method and integration configuration>"
  - type: response_headers
    content: "<CORS headers in integration/gateway responses>"
  - type: preflight_test
    content: "<preflight request/response>"
severity: HIGH
mitigation:
  immediate: "Add missing CORS headers to OPTIONS, method responses, and gateway responses"
  long_term: "Implement consistent CORS configuration across all methods and error responses"
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
