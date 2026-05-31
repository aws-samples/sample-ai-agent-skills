---
title: "D1 — Account-Level Throttling"
description: "Diagnose account-level API Gateway throttling causing 429 errors across all APIs"
status: active
severity: CRITICAL
triggers:
  - "429"
  - "throttling"
  - "rate limit"
  - "too many requests"
  - "account limit"
  - "burst"
owner: devops-agent
objective: "Resolve account-level throttling by identifying high-traffic APIs and implementing throttling controls"
context: "API Gateway has an account-level default of 10,000 requests per second (rps) with 5,000 burst capacity, shared across ALL REST and HTTP APIs in the account and region. A single high-traffic API can consume the entire quota, causing 429 errors on other APIs. This limit can be increased via service quota request."
---

## Phase 1 — Triage

MUST:
- Check Count metric across all APIs: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name Count --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check per-API request count: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name Count --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check 429 error count: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 4XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check current account throttle limit: `aws service-quotas get-service-quota --service-code apigateway --quota-code p-8um73hp3a8`
- Identify which APIs are consuming the most requests

SHOULD:
- Check if any API has stage/method-level throttle overrides: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'methodSettings'`
- Check usage plans for per-client throttling: `aws apigateway get-usage-plans`
- Review traffic patterns for burst vs sustained load

MAY:
- Check CloudTrail for recent throttle limit changes
- Review API Gateway access logs for request patterns

## Phase 2 — Remediate

MUST:
- Identify and address the highest-traffic API consuming account quota
- Implement stage-level throttle overrides to protect other APIs
- Request account limit increase if legitimate traffic exceeds 10,000 rps: `aws service-quotas request-service-quota-increase --service-code apigateway --quota-code p-8um73hp3a8 --desired-value <new-limit>`

SHOULD:
- Implement usage plans with API keys for per-client throttling
- Add caching at the API Gateway stage level to reduce backend calls
- Implement client-side retry with exponential backoff and jitter

MAY:
- Use CloudFront in front of API Gateway for additional caching and DDoS protection
- Implement request queuing with SQS for burst traffic

## Common Issues

- symptoms: "429 errors across multiple APIs simultaneously"
  diagnosis: "Account-level throttle limit (10,000 rps) is shared and being exceeded."
  resolution: "Identify high-traffic API, implement stage throttling, and request limit increase."

- symptoms: "429 errors during traffic spikes despite low average traffic"
  diagnosis: "Burst capacity (5,000) exceeded. Burst is consumed faster than it refills."
  resolution: "Implement client-side throttling, request queuing, or increase account limits."

- symptoms: "429 on one API while others work fine"
  diagnosis: "Stage or method-level throttle override is set too low on the affected API."
  resolution: "Check and adjust stage/method throttle settings."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
root_cause: "account_throttling — <specific_cause>"
evidence:
  - type: request_count
    content: "<per-API and total request counts>"
  - type: throttle_limits
    content: "<account and stage-level throttle settings>"
  - type: error_pattern
    content: "<429 error timing and distribution>"
severity: CRITICAL
mitigation:
  immediate: "Implement stage throttling and request limit increase"
  long_term: "Implement usage plans, caching, and client-side throttling"
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
