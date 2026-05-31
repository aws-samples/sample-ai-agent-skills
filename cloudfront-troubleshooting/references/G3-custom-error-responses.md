---
title: "G3 — Custom Error Response Issues"
description: "Diagnose custom error response configuration and caching issues"
status: active
severity: MEDIUM
triggers:
  - "custom error page"
  - "custom error response"
  - "error page not showing"
  - "stale error page"
  - "error caching"
owner: devops-agent
objective: "Resolve custom error response configuration and caching issues"
context: "CloudFront can return custom error pages for specific HTTP error codes. Custom error responses have their own TTL (Error Caching Minimum TTL, default 10 seconds). They can map one error code to a different response code (e.g., 404 → 200 for SPA routing). Custom error pages are cached separately from normal content."
---

## Phase 1 — Triage

MUST:
- Check custom error responses: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.CustomErrorResponses'`
- Verify the error page path exists at the origin
- Check error caching TTL for each configured error code
- Test the error response: `curl -sI https://<domain>/nonexistent-path`

SHOULD:
- Verify the custom error page is accessible (not itself returning an error)
- Check if the error page is cached with stale content
- Verify response code mapping is correct

MAY:
- Check CloudFront access logs for error response patterns
- Test error page rendering in different browsers

## Phase 2 — Remediate

MUST:
- Ensure custom error page path is valid and accessible at the origin
- Set appropriate Error Caching Minimum TTL (lower for transient errors, higher for permanent)
- Invalidate cached error responses if the error page was updated

SHOULD:
- Use response code mapping for SPA routing (404 → 200 with /index.html)
- Set short TTL for 5xx errors (transient) and longer for 4xx (likely permanent)
- Host error pages on a reliable origin (S3 recommended)

MAY:
- Use Lambda@Edge for dynamic error page generation
- Implement different error pages per cache behavior using Lambda@Edge

## Common Issues

- symptoms: "Custom error page not showing — default CloudFront error displayed"
  diagnosis: "Custom error response not configured for the specific error code."
  resolution: "Add custom error response for the error code in distribution config."

- symptoms: "Error page shows but origin error is resolved — stale error cached"
  diagnosis: "Error Caching Minimum TTL has not expired."
  resolution: "Reduce Error Caching Minimum TTL or invalidate the error response."

- symptoms: "SPA returns 404 on direct URL access"
  diagnosis: "No custom error response mapping 404 to 200 with /index.html."
  resolution: "Add custom error response: 404 → 200, response page path /index.html."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
```

## Escalation Conditions

- Distribution serves a production website or application
- Fix requires modifying origin configuration or cache behaviors
- Resolution involves certificate changes or HTTPS configuration
- Issue affects multiple distributions or is account-level
- Lambda@Edge or CloudFront Functions changes are required on production

## Data Sensitivity

MEDIUM - Signed URL private keys and key pairs control content access. Origin configurations may expose internal infrastructure (S3 bucket names, ALB endpoints). Access logs contain client IPs, request URIs, and query strings. Field-level encryption configurations protect sensitive form data.

## Prohibited Actions

- NEVER suggest deleting a CloudFront distribution that is serving live traffic
- NEVER suggest disabling HTTPS or downgrading the security policy on a production distribution
- NEVER recommend removing all cache behaviors - this breaks content routing
- NEVER suggest invalidating '/*' repeatedly as a fix - address the root caching issue instead
- NEVER recommend removing origin access control/identity from S3 origins without alternative access controls

## Phase 3 - Rollback

1. If distribution configuration was changed, update with previous settings: `aws cloudfront update-distribution --id <id> --distribution-config <previous> --if-match <etag>`
2. If cache policy or TTL was changed, restore original cache behavior settings and allow caches to repopulate
3. If SSL/TLS settings were changed, restore original certificate and security policy configuration
4. If edge function was changed, update the distribution to use the previous function version/ARN
5. If origin configuration was changed, restore original origin settings including timeouts and protocols
6. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "custom_error_response — <specific_cause>"
evidence:
  - type: error_config
    content: "<custom error response configuration>"
  - type: error_ttl
    content: "<error caching TTL>"
  - type: error_page_status
    content: "<error page accessibility>"
severity: MEDIUM
mitigation:
  immediate: "Fix error response configuration or invalidate cached error"
  long_term: "Implement proper error page strategy with appropriate TTLs"
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
  - "NEVER suggest disabling HTTPS requirements"
  - "NEVER suggest removing WAF association to fix access"
  - "NEVER suggest wildcard CORS origins in production"
