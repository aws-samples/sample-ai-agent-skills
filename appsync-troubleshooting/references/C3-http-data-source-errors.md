---
title: "C3 — HTTP Data Source Errors"
description: "Diagnose AppSync HTTP data source connection and response errors"
status: active
severity: HIGH
triggers:
  - "HTTP data source"
  - "HTTP resolver error"
  - "external API error"
  - "HTTP endpoint"
  - "REST API integration"
owner: devops-agent
objective: "Identify and resolve HTTP data source errors in AppSync resolvers"
context: "HTTP data sources allow AppSync to call external REST APIs and HTTP endpoints. Common issues include endpoint unreachability, SSL certificate errors, incorrect request mapping (method, headers, body), response parsing failures, and timeout issues. There is no built-in retry or circuit breaker."
---

## Phase 1 — Triage

MUST:
- Get the data source configuration: `aws appsync get-data-source --api-id <api-id> --name <ds-name>`
- Verify the HTTP endpoint is accessible from AppSync (public endpoint or VPC-configured)
- Check the resolver request mapping for correct HTTP method, path, headers, and body
- Check CloudWatch Logs for HTTP response status codes
- Verify the endpoint URL is correct and includes the protocol (https://)

SHOULD:
- Test the HTTP endpoint independently: `curl -v <endpoint-url>`
- Check if the endpoint requires authentication headers
- Verify the response mapping correctly parses the HTTP response body

MAY:
- Check if the endpoint has rate limiting that AppSync requests might trigger
- Verify SSL certificate validity for HTTPS endpoints

## Phase 2 — Remediate

MUST:
- Fix the endpoint URL (must be HTTPS for production)
- Ensure the request mapping produces the correct HTTP method, path, and headers
- Handle HTTP error responses in the response mapping template

SHOULD:
- Add timeout handling in the resolver (HTTP calls can be slow)
- Implement error handling for non-2xx responses
- Use IAM authorization for AWS service endpoints

MAY:
- Add caching for frequently accessed HTTP endpoints
- Use pipeline resolvers to implement retry logic

## Common Issues

- symptoms: "HTTP data source returns connection refused"
  diagnosis: "Endpoint is unreachable from AppSync. May be a private endpoint without VPC configuration."
  resolution: "Ensure the endpoint is publicly accessible or configure AppSync with VPC access."

- symptoms: "HTTP resolver returns 403 Forbidden"
  diagnosis: "Endpoint requires authentication that isn't configured in the request mapping."
  resolution: "Add required authentication headers in the request mapping template."

- symptoms: "Response mapping fails to parse HTTP response"
  diagnosis: "Response body format doesn't match the parsing logic in the response mapping."
  resolution: "Check the actual response format and update the response mapping accordingly."

## Output Format

```yaml
root_cause: "http_data_source_error — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<HTTP endpoint URL and configuration>"
  - type: resolver_logs
    content: "<HTTP response status and body>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint URL, request mapping, or authentication"
  long_term: "Add error handling, caching, and monitoring for HTTP endpoints"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix endpoint URL | GREEN | Configuration fix |
| Ensure request mapping correct | YELLOW | Resolver code change |
| Handle HTTP error responses | YELLOW | Resolver code change |
| Add timeout handling | YELLOW | Resolver code change |
| Implement error handling for non-2xx | YELLOW | Resolver code change |
| Use IAM auth for AWS endpoints | YELLOW | Auth configuration change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **HTTP endpoint URL**: External API location — INTERNAL
- **Request mapping**: HTTP method, headers, body — INTERNAL
- **Response data**: External API response content — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER hardcode API keys or secrets in resolver templates
- NEVER use HTTP (non-TLS) endpoints in production

## Phase 3 — Rollback

1. Revert resolver request/response mapping templates
2. Revert data source endpoint URL if changed
3. Revert IAM authorization configuration if modified
4. Verify resolver returns expected results
5. Monitor HTTP data source error rates after rollback

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
