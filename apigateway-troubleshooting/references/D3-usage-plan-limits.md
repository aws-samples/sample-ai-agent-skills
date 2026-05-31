---
title: "D3 — Usage Plan Limits"
description: "Diagnose usage plan throttling and quota issues for API key-based clients"
status: active
severity: HIGH
triggers:
  - "usage plan"
  - "API key"
  - "quota"
  - "quota exceeded"
  - "usage limit"
  - "x-api-key"
owner: devops-agent
objective: "Resolve usage plan throttling and quota issues by analyzing plan configuration, key associations, and consumption"
context: "Usage plans associate API keys with throttle limits (rps/burst) and quotas (requests per day/week/month). API keys are sent in the x-api-key header. Usage plans provide per-client rate limiting. Quotas reset at the beginning of each period. API keys are for identification and throttling, NOT authentication."
---

## Phase 1 — Triage

MUST:
- Check usage plan configuration: `aws apigateway get-usage-plan --usage-plan-id <plan-id>`
- Check API key usage: `aws apigateway get-usage --usage-plan-id <plan-id> --key-id <key-id> --start-date <YYYY-MM-DD> --end-date <YYYY-MM-DD>`
- List usage plan keys: `aws apigateway get-usage-plan-keys --usage-plan-id <plan-id>`
- Verify API key is associated with the correct usage plan
- Check if method requires API key: `aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'apiKeyRequired'`

SHOULD:
- Check usage plan throttle and quota settings: `aws apigateway get-usage-plan --usage-plan-id <plan-id> --query '{throttle:throttle,quota:quota}'`
- Check API stages associated with the usage plan: `aws apigateway get-usage-plan --usage-plan-id <plan-id> --query 'apiStages'`
- Verify the API key is enabled: `aws apigateway get-api-key --api-key <key-id> --query 'enabled'`

MAY:
- Check all usage plans for the API: `aws apigateway get-usage-plans --query 'items[*].{id:id,name:name,throttle:throttle,quota:quota}'`
- Review usage patterns over time to right-size limits

## Phase 2 — Remediate

MUST:
- Increase quota if legitimately exceeded: `aws apigateway update-usage --usage-plan-id <plan-id> --key-id <key-id> --patch-operations op=replace,path=/remaining,value=<amount>`
- Adjust usage plan throttle/quota limits if too restrictive
- Ensure API key is enabled and associated with the correct usage plan

SHOULD:
- Create tiered usage plans (free, basic, premium) with different limits
- Monitor usage plan consumption with CloudWatch
- Implement client notification before quota exhaustion

MAY:
- Implement usage plan extensions for temporary limit increases
- Use multiple API keys per client for different use cases

## Common Issues

- symptoms: "429 with 'Limit Exceeded' for specific API key"
  diagnosis: "Usage plan quota (daily/weekly/monthly) is exhausted."
  resolution: "Increase quota limit or grant additional usage for the current period."

- symptoms: "403 Forbidden with 'x-api-key' header present"
  diagnosis: "API key is not associated with a usage plan that includes the API stage."
  resolution: "Associate the API key with a usage plan that includes the target API stage."

- symptoms: "429 for one client but not others"
  diagnosis: "Per-client throttle limit in usage plan is exceeded for that API key."
  resolution: "Increase per-client throttle in the usage plan or optimize client request patterns."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
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
4. If custom domain or base path mapping was changed, restore original mapping configuration
5. If CORS or method configuration was changed, restore original settings and redeploy the API
6. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "usage_plan — <specific_cause>"
evidence:
  - type: usage_plan
    content: "<usage plan throttle and quota settings>"
  - type: key_usage
    content: "<API key consumption data>"
  - type: association
    content: "<API key to usage plan to API stage mapping>"
severity: HIGH
mitigation:
  immediate: "Adjust usage plan limits or grant additional quota"
  long_term: "Implement tiered usage plans and consumption monitoring"
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
