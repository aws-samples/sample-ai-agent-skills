---
title: "A2 — Lambda Custom Integration Issues"
description: "Diagnose Lambda custom (non-proxy) integration failures including mapping template errors"
status: active
severity: HIGH
triggers:
  - "custom integration"
  - "mapping template"
  - "VTL"
  - "request template"
  - "response template"
  - "integration response"
owner: devops-agent
objective: "Resolve Lambda custom integration issues including VTL mapping template errors and integration request/response configuration"
context: "Lambda custom integration (type AWS) uses mapping templates to transform requests before sending to Lambda and responses before returning to the client. Unlike proxy integration, the request/response transformation is handled by API Gateway using Apache Velocity Template Language (VTL). Errors in mapping templates cause 500 errors."
---

## Phase 1 — Triage

MUST:
- Verify integration type is AWS (not AWS_PROXY): `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method>`
- Check request mapping template: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'requestTemplates'`
- Check integration response mappings: `aws apigateway get-integration-response --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --status-code 200`
- Test invoke to see transformation results: `aws apigateway test-invoke-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --body '{"key":"value"}'`
- Check execution logs for template errors: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --start-time <epoch-ms>`

SHOULD:
- Check method response models: `aws apigateway get-method-response --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --status-code 200`
- Verify content type handling and passthrough behavior: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'passthroughBehavior'`
- Check if Lambda returns expected format for the response template

MAY:
- Review all integration response status code mappings
- Check for regex patterns in integration response selection

## Phase 2 — Remediate

MUST:
- Fix VTL syntax errors in mapping templates
- Ensure request template produces valid JSON for Lambda input
- Ensure response template correctly maps Lambda output to API response
- Verify passthrough behavior setting (WHEN_NO_MATCH, WHEN_NO_TEMPLATES, NEVER)

SHOULD:
- Add error response mappings for Lambda error patterns (selectionPattern regex)
- Use $input.json('$') for simple passthrough of JSON body
- Map common error patterns to appropriate HTTP status codes

MAY:
- Use $context variables for request metadata (requestId, identity, etc.)
- Implement response caching for frequently accessed data

## Common Issues

- symptoms: "500 error with 'Execution failed due to configuration error' in logs"
  diagnosis: "VTL mapping template has syntax errors or references undefined variables."
  resolution: "Fix VTL template syntax. Use $input.body for raw body, $input.json('$.key') for specific fields."

- symptoms: "Empty response body despite Lambda returning data"
  diagnosis: "Response mapping template does not map Lambda output correctly."
  resolution: "Check integration response template. Use $input.json('$') to pass through Lambda response."

- symptoms: "Request body not reaching Lambda"
  diagnosis: "Request mapping template is empty or passthroughBehavior is NEVER with no matching template."
  resolution: "Add request template for the content type or set passthroughBehavior to WHEN_NO_MATCH."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
3. If custom domain or base path mapping was changed, restore original mapping configuration
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "custom_integration — <specific_cause>"
evidence:
  - type: request_template
    content: "<VTL request mapping template>"
  - type: response_template
    content: "<VTL response mapping template>"
  - type: execution_logs
    content: "<template execution errors>"
severity: HIGH
mitigation:
  immediate: "Fix mapping template syntax or configuration"
  long_term: "Consider migrating to Lambda proxy integration for simpler setup"
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
