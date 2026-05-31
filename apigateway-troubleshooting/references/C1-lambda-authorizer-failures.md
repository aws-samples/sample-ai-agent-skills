---
title: "C1 — Lambda Authorizer Failures"
description: "Diagnose Lambda authorizer (custom authorizer) failures including caching, policy format, and execution errors"
status: active
severity: HIGH
triggers:
  - "Lambda authorizer"
  - "custom authorizer"
  - "authorization"
  - "401"
  - "403"
  - "Unauthorized"
  - "authorizer"
  - "token"
owner: devops-agent
objective: "Resolve Lambda authorizer failures by validating authorizer configuration, Lambda function, policy output, and caching behavior"
context: "Lambda authorizers (TOKEN or REQUEST type) invoke a Lambda function to authorize API requests. The function must return an IAM policy document. TOKEN authorizers use a single header value. REQUEST authorizers use headers, query strings, stage variables, and context. Results can be cached (TTL 0-3600s). Common failures include malformed policy output, Lambda errors, and stale cached decisions."
---

## Phase 1 — Triage

MUST:
- Check authorizer configuration: `aws apigateway get-authorizer --rest-api-id <api-id> --authorizer-id <auth-id>`
- Check authorizer type (TOKEN vs REQUEST) and identity sources
- Check authorizer Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<authorizer-function> --start-time <epoch-ms>`
- Check execution logs for authorizer errors: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "Authorizer" --start-time <epoch-ms>`
- Check authorizer cache TTL: `aws apigateway get-authorizer --rest-api-id <api-id> --authorizer-id <auth-id> --query 'authorizerResultTtlInSeconds'`
- Verify Lambda has permission to be invoked by API Gateway: `aws lambda get-policy --function-name <authorizer-function>`

SHOULD:
- Test authorizer directly: `aws apigateway test-invoke-authorizer --rest-api-id <api-id> --authorizer-id <auth-id> --headers Authorization=<token>`
- Check if the issue is caused by cached results (set TTL to 0 to test)
- Verify the authorizer Lambda returns a valid IAM policy document

MAY:
- Check Lambda timeout (should be well under 10 seconds for authorizers)
- Review authorizer Lambda concurrency limits

## Phase 2 — Remediate

MUST:
- Ensure authorizer Lambda returns valid policy format:
  ```json
  {
    "principalId": "user123",
    "policyDocument": {
      "Version": "2012-10-17",
      "Statement": [{
        "Action": "execute-api:Invoke",
        "Effect": "Allow",
        "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/<stage>/<method>/<path>"
      }]
    },
    "context": { "key": "value" }
  }
  ```
- Fix Lambda function errors and ensure it handles all token formats
- Set TTL to 0 during debugging to bypass cache

SHOULD:
- Use wildcard resources in policy for broader access: `arn:aws:execute-api:*:*:*`
- Add context variables for downstream use in mapping templates
- Implement proper error handling — return "Unauthorized" string for 401, or "Deny" policy for 403

MAY:
- Implement token validation (JWT verification) in the authorizer
- Use REQUEST type authorizer for multi-source identity validation

## Common Issues

- symptoms: "401 Unauthorized even with valid token"
  diagnosis: "Authorizer Lambda throws an error or returns 'Unauthorized' string."
  resolution: "Check authorizer Lambda logs. Ensure it returns a valid policy, not an error."

- symptoms: "403 after updating authorizer but old behavior persists"
  diagnosis: "Authorizer result is cached. Old Allow/Deny decision is still in effect."
  resolution: "Set authorizer TTL to 0 to disable caching, or wait for cache expiry."

- symptoms: "500 error with 'Authorizer error' in execution logs"
  diagnosis: "Authorizer Lambda returns malformed policy document."
  resolution: "Validate policy format: principalId, policyDocument with Version and Statement."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
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
4. If custom domain or base path mapping was changed, restore original mapping configuration
5. If CORS or method configuration was changed, restore original settings and redeploy the API
6. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "lambda_authorizer — <specific_cause>"
evidence:
  - type: authorizer_config
    content: "<authorizer type, identity sources, TTL>"
  - type: authorizer_output
    content: "<Lambda authorizer response/policy>"
  - type: execution_logs
    content: "<authorizer-related execution log entries>"
severity: HIGH
mitigation:
  immediate: "Fix authorizer Lambda output format or disable caching for debugging"
  long_term: "Implement robust token validation and proper error handling in authorizer"
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
