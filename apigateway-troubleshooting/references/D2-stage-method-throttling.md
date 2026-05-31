---
title: "D2 — Stage and Method-Level Throttling"
description: "Diagnose stage-level and method-level throttling configuration issues"
status: active
severity: HIGH
triggers:
  - "stage throttling"
  - "method throttling"
  - "per-method limit"
  - "throttle override"
  - "rate limit per method"
owner: devops-agent
objective: "Resolve stage and method-level throttling issues by analyzing throttle hierarchy and configuration"
context: "Stage-level and method-level throttle overrides allow fine-grained rate limiting. These overrides can only REDUCE limits below the account default — they cannot exceed account limits. Method settings use the format 'resource-path/HTTP-method' (e.g., '~1users/GET'). The throttle hierarchy is: account > stage > method. Usage plans add per-client limits on top."
---

## Phase 1 — Triage

MUST:
- Check stage method settings: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'methodSettings'`
- Check default stage throttle (*/*): `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'methodSettings."*/*"'`
- Check specific method overrides: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'methodSettings."~1<path>/<METHOD>"'`
- Check 429 error rate per method: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 4XXError --dimensions Name=ApiName,Value=<api-name> Name=Method,Value=<method> Name=Resource,Value=<path> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Verify account-level limits are not the bottleneck

SHOULD:
- Compare configured limits with actual traffic patterns
- Check if usage plans impose additional per-client limits
- Review all method settings for unintended low limits

MAY:
- Check CloudTrail for recent throttle setting changes
- Analyze traffic patterns to determine appropriate limits

## Phase 2 — Remediate

MUST:
- Adjust method-level throttle settings: `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=replace,path=/~1users~1GET/throttling/rateLimit,value=1000 op=replace,path=/~1users~1GET/throttling/burstLimit,value=500`
- Ensure method limits are appropriate for expected traffic
- Create a new deployment if method settings were changed via API configuration

SHOULD:
- Set stage default (*/*) throttle as a baseline
- Override specific high-traffic methods with higher limits
- Override sensitive methods (writes, deletes) with lower limits

MAY:
- Implement per-client throttling via usage plans for multi-tenant APIs
- Add CloudWatch alarms for throttle events

## Common Issues

- symptoms: "429 on specific method but other methods work"
  diagnosis: "Method-level throttle override is set too low for that specific method."
  resolution: "Check and increase the method-level throttle setting."

- symptoms: "429 after stage configuration change"
  diagnosis: "Default stage throttle (*/*) was set too low, affecting all methods."
  resolution: "Increase the default stage throttle or set method-specific overrides."

- symptoms: "Throttle settings appear correct but 429 persists"
  diagnosis: "Account-level limit is the bottleneck, not stage/method limits."
  resolution: "Check account-level limits. Stage/method limits cannot exceed account limits."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "stage_method_throttling — <specific_cause>"
evidence:
  - type: method_settings
    content: "<stage and method throttle configuration>"
  - type: traffic_pattern
    content: "<actual request rate vs configured limits>"
  - type: throttle_hierarchy
    content: "<account, stage, and method limit comparison>"
severity: HIGH
mitigation:
  immediate: "Adjust stage or method-level throttle settings"
  long_term: "Implement throttle hierarchy aligned with traffic patterns"
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
