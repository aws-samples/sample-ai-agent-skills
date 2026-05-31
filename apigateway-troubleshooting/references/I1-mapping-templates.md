---
title: "I1 — Mapping Template Issues"
description: "Diagnose VTL mapping template errors in request/response transformation"
status: active
severity: MEDIUM
triggers:
  - "mapping template"
  - "VTL"
  - "Velocity"
  - "request template"
  - "response template"
  - "$input"
  - "$context"
  - "transformation"
owner: devops-agent
objective: "Resolve VTL mapping template errors by validating template syntax, variable references, and transformation logic"
context: "Mapping templates use Apache Velocity Template Language (VTL) to transform request/response payloads in REST API custom integrations. Templates have access to $input (request body, params), $context (request context), $stageVariables, and $util (utility functions). Template errors cause 500 Internal Server Error. Templates are NOT available in Lambda proxy integration or HTTP API."
---

## Phase 1 — Triage

MUST:
- Check request mapping template: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'requestTemplates'`
- Check response mapping template: `aws apigateway get-integration-response --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --status-code 200 --query 'responseTemplates'`
- Test invoke to see template output: `aws apigateway test-invoke-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --body '{"test":"data"}'`
- Check execution logs for template errors: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "Execution failed" --start-time <epoch-ms>`
- Verify integration type is AWS (custom), not AWS_PROXY

SHOULD:
- Check passthrough behavior: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'passthroughBehavior'`
- Verify content type mapping (application/json is most common)
- Check for undefined variable references in templates

MAY:
- Test template logic with sample payloads
- Review $util helper functions usage

## Phase 2 — Remediate

MUST:
- Fix VTL syntax errors (missing #end, unclosed quotes, invalid references)
- Use correct variable access: `$input.json('$.key')` for JSON fields, `$input.body` for raw body
- Ensure template produces valid JSON output
- Handle null values: `#if($input.json('$.key') != "") ... #end`

SHOULD:
- Use `$util.escapeJavaScript()` for string values to prevent JSON injection
- Use `$input.params('paramName')` for path/query/header parameters
- Set passthrough behavior to WHEN_NO_MATCH for flexibility

MAY:
- Use `$util.base64Encode()` / `$util.base64Decode()` for binary data
- Implement error response templates with selectionPattern regex
- Use `$context.requestId` for request correlation

## Common Issues

- symptoms: "500 error with 'Execution failed due to configuration error'"
  diagnosis: "VTL template has syntax error or references undefined variable."
  resolution: "Check template syntax. Use $input.json('$') for simple passthrough. Fix #if/#end blocks."

- symptoms: "Request body not reaching backend"
  diagnosis: "No request template for the content type and passthroughBehavior is NEVER."
  resolution: "Add template for application/json or set passthroughBehavior to WHEN_NO_MATCH."

- symptoms: "Response body is empty or malformed"
  diagnosis: "Response template does not correctly map integration response."
  resolution: "Use $input.json('$') in response template to pass through backend response."


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
root_cause: "mapping_template — <specific_cause>"
evidence:
  - type: request_template
    content: "<VTL request template content>"
  - type: response_template
    content: "<VTL response template content>"
  - type: test_invoke
    content: "<test invoke result showing transformation>"
severity: MEDIUM
mitigation:
  immediate: "Fix VTL template syntax or variable references"
  long_term: "Consider Lambda proxy integration for simpler request/response handling"
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
