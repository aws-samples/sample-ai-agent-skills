---
title: "C2 — Viewer Response Errors"
description: "Diagnose viewer-response event function errors"
status: active
severity: MEDIUM
triggers: ["viewer response error", "response headers wrong", "missing security headers", "viewer-response function failed"]
owner: devops-agent
objective: "Resolve viewer-response event function errors"
context: "Viewer-response functions run after CloudFront retrieves the response (from cache or origin) and before sending to the viewer. Common use: adding security headers."
---

## Phase 1 — Triage

MUST:
- Identify the associated function: `aws cloudfront get-distribution-config --id <distribution-id>`
- Check function execution errors: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionExecutionErrors --dimensions Name=DistributionId,Value=<dist-id>,Name=FunctionName,Value=<func-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify the function returns a valid response object

SHOULD:
- Check if the function modifies headers correctly (Content-Type, security headers)
- Verify the function does not remove required headers
- Test with different response status codes (200, 301, 404, 500)

MAY:
- Compare response headers with and without the function
- Check if caching affects the function's behavior

## Phase 2 — Remediate

MUST:
- Fix function code to return a valid response object
- Ensure required headers are preserved (Content-Type, Content-Length)
- Handle all response status codes gracefully

SHOULD:
- Add security headers consistently (Strict-Transport-Security, X-Content-Type-Options, etc.)
- Test with cached and uncached responses
- Verify the function works with both CF Functions and Lambda@Edge response formats

MAY:
- Implement response header policies as an alternative to functions
- Use CloudFront response headers policy for standard security headers

## Common Issues

- symptoms: "Security headers missing from some responses"
  diagnosis: "Function only runs on cache misses, not cache hits (if using Lambda@Edge origin-response instead of viewer-response)."
  resolution: "Use viewer-response event to add headers to ALL responses, including cached ones."

- symptoms: "Response function causes 502 errors"
  diagnosis: "Function throws exception or returns invalid response object."
  resolution: "Add try-catch. Ensure the function always returns the response object, even if header manipulation fails."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "cloudwatch get-metric-statistics: GREEN — read-only"
  - "cloudfront test-function: GREEN — no production impact"
  - "cloudfront update-distribution: YELLOW — modifies response behavior"
```

## Escalation Conditions

- Viewer-response errors cause incorrect content delivery
- Security headers are missing, creating compliance violations
- Function removes critical headers causing downstream client failures
- Response manipulation conflicts with origin-set headers

## Data Sensitivity

- `aws cloudfront get-distribution-config`: MEDIUM — reveals distribution settings
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics
- Response headers: MEDIUM — may contain security configuration details
- `aws cloudfront test-function`: MEDIUM — test events contain response data

## Prohibited Actions

- NEVER remove Content-Type or Content-Length headers from responses
- NEVER add headers that expose internal infrastructure details (Server, X-Powered-By)
- NEVER modify response bodies in viewer-response functions (not supported)
- NEVER set overly permissive CORS headers without security review

## Phase 3 — Rollback

1. Disassociate the viewer-response function from the cache behavior
2. Or publish previous working version of the function
3. Verify response headers are correct after rollback
4. Check if CloudFront response headers policy can replace the function
5. Document the response handling requirements

## Output Format

```yaml
root_cause: "viewer_response — <specific_cause>"
evidence:
  - type: response_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix response function or rollback"
  long_term: "Use CloudFront response headers policy where possible"
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
