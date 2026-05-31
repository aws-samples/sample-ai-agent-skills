---
title: "I2 — Request Validation Issues"
description: "Diagnose request validation failures including body, query string, and header validation"
status: active
severity: MEDIUM
triggers:
  - "request validation"
  - "validator"
  - "JSON schema"
  - "model"
  - "invalid request"
  - "request body"
  - "required parameter"
owner: devops-agent
objective: "Resolve request validation issues by analyzing validators, models, and required parameter configuration"
context: "REST API supports request validation for body (using JSON Schema models), query string parameters, and headers. Validators are configured per method and can validate body only, parameters only, or both. Validation failures return 400 Bad Request with a message indicating what failed. HTTP API does not have built-in request validation — use Lambda or middleware."
---

## Phase 1 — Triage

MUST:
- Check request validators: `aws apigateway get-request-validators --rest-api-id <api-id>`
- Check method validator: `aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query '{validator:requestValidatorId,models:requestModels,params:requestParameters}'`
- Check model definition: `aws apigateway get-model --rest-api-id <api-id> --model-name <model-name>`
- Test with invalid request to see validation error: `aws apigateway test-invoke-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --body '{"invalid":"data"}'`
- Check execution logs for validation errors: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "Invalid request body" --start-time <epoch-ms>`

SHOULD:
- Verify JSON Schema model is valid
- Check required parameters are marked correctly in method request
- Verify content type matches the model mapping

MAY:
- List all models: `aws apigateway get-models --rest-api-id <api-id>`
- Check if validation is too strict or too lenient

## Phase 2 — Remediate

MUST:
- Fix JSON Schema model to match expected request format
- Ensure validator type matches what needs validation (body, parameters, or both)
- Associate validator with the method: `aws apigateway update-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --patch-operations op=replace,path=/requestValidatorId,value=<validator-id>`
- Create deployment after changes

SHOULD:
- Use descriptive validation error messages in gateway responses
- Define models for all request bodies
- Mark required query string and header parameters

MAY:
- Implement custom gateway response for validation failures with CORS headers
- Use $ref in JSON Schema for reusable model components

## Common Issues

- symptoms: "400 Bad Request with 'Invalid request body'"
  diagnosis: "Request body does not match the JSON Schema model."
  resolution: "Check model schema against the request. Fix required fields, types, or patterns."

- symptoms: "Validation not working — invalid requests pass through"
  diagnosis: "No request validator associated with the method."
  resolution: "Create a validator and associate it with the method. Deploy changes."

- symptoms: "400 error on valid request"
  diagnosis: "JSON Schema model is too restrictive or has additionalProperties: false."
  resolution: "Review and relax model schema. Allow additional properties if needed."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
root_cause: "request_validation — <specific_cause>"
evidence:
  - type: validator
    content: "<validator configuration>"
  - type: model
    content: "<JSON Schema model definition>"
  - type: validation_error
    content: "<specific validation error message>"
severity: MEDIUM
mitigation:
  immediate: "Fix model schema or validator configuration"
  long_term: "Implement comprehensive request validation with clear error messages"
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
