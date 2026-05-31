---
title: "D2 — Origin Response Errors"
description: "Diagnose origin-response event Lambda@Edge errors"
status: active
severity: MEDIUM
triggers: ["origin response error", "Lambda@Edge origin-response failed", "response modification error", "cache poisoning"]
owner: devops-agent
objective: "Resolve origin-response Lambda@Edge function errors"
context: "Origin-response functions run after CloudFront receives the response from the origin and before caching. Only Lambda@Edge supports this event. 30-second timeout."
---

## Phase 1 — Triage

MUST:
- Verify Lambda@Edge function is associated with origin-response: `aws cloudfront get-distribution-config --id <distribution-id>`
- Check Lambda@Edge logs: `aws logs filter-log-events --log-group-name /aws/lambda/us-east-1.<function-name> --start-time <epoch-ms> --filter-pattern "ERROR" --region <edge-region>`
- Check if the function modifies response headers or status codes correctly

SHOULD:
- Verify the function returns a valid response object
- Check if modified responses are being cached incorrectly
- Test with different origin response status codes (200, 404, 500)

MAY:
- Compare cached responses with origin responses to detect modifications
- Check if the function generates response bodies (40KB limit for origin events)

## Phase 2 — Remediate

MUST:
- Fix function code to return a valid response object
- Ensure Cache-Control and Vary headers are set correctly to prevent cache poisoning
- Handle all origin response status codes gracefully

SHOULD:
- Add error handling that passes through the original response on failure
- Verify response body generation stays within 40KB limit
- Test with error responses from the origin (4xx, 5xx)

MAY:
- Consider using viewer-response for header additions (runs on all responses, not just cache misses)
- Implement response validation before caching

## Common Issues

- symptoms: "Incorrect responses cached and served to all users"
  diagnosis: "Origin-response function modifies response without setting appropriate Vary header."
  resolution: "Add Vary header for any request-dependent response modifications. Invalidate poisoned cache entries."

- symptoms: "Origin-response function causes 502 on cache misses"
  diagnosis: "Function throws exception processing the origin response."
  resolution: "Add try-catch. Return the original response object on error."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "logs filter-log-events: GREEN — read-only"
  - "cloudfront create-invalidation: YELLOW — clears cached content"
  - "cloudfront update-distribution: RED — modifies response processing"
```

## Escalation Conditions

- Cache poisoning has served incorrect content to users
- Origin-response function causes intermittent 502 errors
- Response body generation exceeds 40KB limit for required use case
- Function processing time approaches 30-second timeout

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — may contain response data and headers
- `aws cloudfront get-distribution-config`: MEDIUM — reveals distribution settings
- Origin response data: HIGH — may contain sensitive content
- Cached response data: HIGH — may contain user-specific data if improperly cached

## Prohibited Actions

- NEVER cache user-specific responses without proper Vary headers
- NEVER generate response bodies exceeding 40KB in origin-response events
- NEVER remove Cache-Control headers set by the origin without understanding implications
- NEVER modify Set-Cookie headers without security review

## Phase 3 — Rollback

1. Remove or replace the origin-response function in the distribution
2. Invalidate the CloudFront cache to clear any poisoned entries: `aws cloudfront create-invalidation --distribution-id <dist-id> --paths "/*"`
3. Wait for distribution to reach 'Deployed' status
4. Verify origin responses are cached correctly after rollback
5. Document the cache poisoning risk and prevention measures

## Output Format

```yaml
root_cause: "origin_response — <specific_cause>"
evidence:
  - type: response_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix function or rollback, invalidate cache"
  long_term: "Implement response validation and proper cache key design"
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
