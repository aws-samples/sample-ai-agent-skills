---
title: "D1 — Origin Request Errors"
description: "Diagnose origin-request event Lambda@Edge errors"
status: active
severity: HIGH
triggers: ["origin request error", "Lambda@Edge origin-request failed", "origin routing error", "cache miss function error"]
owner: devops-agent
objective: "Resolve origin-request Lambda@Edge function errors"
context: "Origin-request functions run on cache misses before CloudFront forwards the request to the origin. Only Lambda@Edge supports this event type (not CF Functions). 30-second timeout."
---

## Phase 1 — Triage

MUST:
- Verify Lambda@Edge function is associated with origin-request: `aws cloudfront get-distribution-config --id <distribution-id>`
- Check Lambda@Edge logs in the executing region: `aws logs filter-log-events --log-group-name /aws/lambda/us-east-1.<function-name> --start-time <epoch-ms> --filter-pattern "ERROR" --region <edge-region>`
- Check function error metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=us-east-1.<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum --region <edge-region>`

SHOULD:
- Verify the function returns a valid request object (to forward to origin) or response object (to return directly)
- Check if the function modifies the origin, host header, or URI correctly
- Verify network calls from the function succeed (DNS, HTTP to external services)

MAY:
- Test the function locally with a sample origin-request event
- Check if the error correlates with specific URI patterns or origins

## Phase 2 — Remediate

MUST:
- Fix function code errors identified in logs
- Ensure the function returns either a valid request (to forward) or response (to return)
- Verify origin hostname and path modifications are correct

SHOULD:
- Add error handling that returns a meaningful error response instead of 502
- Implement timeout handling for network calls (stay within 30s limit)
- Cache external service responses to reduce latency and failure risk

MAY:
- Implement origin failover logic in the function
- Add request logging for debugging (redact sensitive data)

## Common Issues

- symptoms: "Origin-request function causes 502 for all cache misses"
  diagnosis: "Function throws unhandled exception or returns invalid request/response object."
  resolution: "Add try-catch. Return original request object on error to fall through to origin."

- symptoms: "Function modifies Host header incorrectly, causing origin 403"
  diagnosis: "Host header doesn't match the origin's expected domain."
  resolution: "Set Host header to match the origin domain name when changing the origin."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "logs filter-log-events: GREEN — read-only"
  - "cloudwatch get-metric-statistics: GREEN — read-only"
  - "cloudfront update-distribution: RED — modifies origin request behavior"
```

## Escalation Conditions

- Origin-request errors cause all cache misses to fail
- Function requires network calls to services that are unreliable
- Origin routing logic is complex and difficult to debug
- Function timeout (30s) is insufficient for required processing

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — may contain request URIs, headers, and origin details
- `aws cloudfront get-distribution-config`: MEDIUM — reveals origin configuration
- Origin request data: HIGH — may contain authentication headers and query parameters
- Network call responses: HIGH — may contain sensitive data from external services

## Prohibited Actions

- NEVER use CloudFront Functions for origin-request events — only Lambda@Edge is supported
- NEVER modify the origin to an untrusted server
- NEVER forward authentication headers to unintended origins
- NEVER make synchronous network calls without timeout handling

## Phase 3 — Rollback

1. Update distribution to remove or replace the origin-request function
2. Wait for distribution to reach 'Deployed' status
3. Verify cache misses correctly reach the origin after rollback
4. Invalidate CloudFront cache if incorrect responses were cached: `aws cloudfront create-invalidation --distribution-id <dist-id> --paths "/*"`
5. Document the origin-request function failure

## Output Format

```yaml
root_cause: "origin_request — <specific_cause>"
evidence:
  - type: function_error
    content: "<error details from logs>"
severity: HIGH
mitigation:
  immediate: "Fix function or remove association"
  long_term: "Implement robust error handling and origin failover"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
