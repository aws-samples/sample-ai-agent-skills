---
title: "E2 — Stage Configuration Issues"
description: "Diagnose stage configuration problems including variables, caching, logging, and settings"
status: active
severity: MEDIUM
triggers:
  - "stage"
  - "stage variable"
  - "caching"
  - "stage settings"
  - "method settings"
  - "cache"
owner: devops-agent
objective: "Resolve stage configuration issues including stage variables, caching, logging settings, and method overrides"
context: "API Gateway stages represent deployment snapshots with additional configuration: stage variables, caching, logging, throttling, and method-level overrides. Stage variables can be used in integration URIs and mapping templates. Caching reduces backend calls but requires careful TTL and key management. Misconfigured stages cause unexpected behavior across environments."
---

## Phase 1 — Triage

MUST:
- Get full stage configuration: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage>`
- Check stage variables: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'variables'`
- Check method settings (caching, logging, throttling): `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'methodSettings'`
- Check cache cluster status: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query '{cacheEnabled:cacheClusterEnabled,cacheSize:cacheClusterSize,cacheStatus:cacheClusterStatus}'`
- Verify stage is associated with correct deployment

SHOULD:
- Check if stage variables are referenced in integrations: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'uri'`
- Verify logging configuration: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'accessLogSettings'`
- Check canary settings if applicable

MAY:
- Compare stage configurations across environments (dev, staging, prod)
- Check CloudWatch for cache hit/miss metrics

## Phase 2 — Remediate

MUST:
- Fix stage variable values for the target environment
- Ensure cache cluster is in AVAILABLE status if caching is enabled
- Verify method-level settings override defaults correctly

SHOULD:
- Use stage variables for environment-specific Lambda aliases or backend URLs
- Configure appropriate cache TTL per method
- Enable execution logging for debugging

MAY:
- Implement cache invalidation strategy
- Use stage-level WAF association for environment-specific protection

## Common Issues

- symptoms: "Integration points to wrong Lambda function/backend in a stage"
  diagnosis: "Stage variable used in integration URI has wrong value for this stage."
  resolution: "Update stage variable: aws apigateway update-stage --rest-api-id <id> --stage-name <stage> --patch-operations op=replace,path=/variables/<key>,value=<value>"

- symptoms: "Caching enabled but responses are always fresh"
  diagnosis: "Cache cluster is not in AVAILABLE status or method-level caching is disabled."
  resolution: "Check cache cluster status. Enable caching per method in method settings."

- symptoms: "Stale responses despite backend changes"
  diagnosis: "API Gateway cache is serving old responses."
  resolution: "Flush stage cache: aws apigateway flush-stage-cache --rest-api-id <id> --stage-name <stage>"


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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
2. If throttling settings were modified, restore original rate/burst limits: `aws apigateway update-stage --rest-api-id <id> --stage-name <stage> --patch-operations <revert>`
3. If integration settings were changed, restore original integration configuration and redeploy
4. If custom domain or base path mapping was changed, restore original mapping configuration
5. If CORS or method configuration was changed, restore original settings and redeploy the API
6. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "stage_config — <specific_cause>"
evidence:
  - type: stage_settings
    content: "<stage variables, caching, logging configuration>"
  - type: method_settings
    content: "<method-level overrides>"
  - type: cache_status
    content: "<cache cluster status and metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix stage variables, caching, or method settings"
  long_term: "Implement consistent stage configuration management across environments"
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
