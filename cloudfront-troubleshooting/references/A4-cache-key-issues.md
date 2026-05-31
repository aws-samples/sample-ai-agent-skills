---
title: "A4 — CloudFront Cache Key Issues"
description: "Diagnose cache key misconfiguration causing cache fragmentation or incorrect content serving"
status: active
severity: MEDIUM
triggers:
  - "cache key"
  - "wrong content served"
  - "cache fragmentation"
  - "query string caching"
  - "header-based caching"
owner: devops-agent
objective: "Optimize cache key configuration to maximize cache hit ratio while serving correct content"
context: "The cache key determines what CloudFront uses to identify cached objects. It can include URL path, query strings, headers, and cookies. Including too many components fragments the cache (low hit ratio). Including too few serves wrong content to different users. Cache policies and origin request policies control this independently."
---

## Phase 1 — Triage

MUST:
- Get the cache policy for the behavior: `aws cloudfront get-cache-policy --id <cache-policy-id>`
- Get the origin request policy: `aws cloudfront get-origin-request-policy --id <origin-request-policy-id>`
- Check which query strings are in the cache key vs forwarded to origin
- Check which headers are in the cache key vs forwarded to origin
- Check which cookies are in the cache key vs forwarded to origin

SHOULD:
- Test with different query string combinations to see if different cached versions are returned
- Check if Accept-Encoding is in the cache key (needed for compression)
- Verify the cache key matches what the origin uses to vary responses

MAY:
- Analyze access logs to identify cache key patterns causing fragmentation
- Check if the origin sends Vary headers that affect caching

## Phase 2 — Remediate

MUST:
- Include only query strings, headers, and cookies that actually change the origin response in the cache key
- Use allowlists instead of forwarding all query strings/headers/cookies
- Ensure Accept-Encoding is included if compression is enabled

SHOULD:
- Use separate cache behaviors for static (minimal cache key) and dynamic (broader cache key) content
- Use managed cache policies (CachingOptimized, CachingDisabled) where appropriate
- Forward additional values to origin via origin request policy WITHOUT adding them to the cache key

MAY:
- Use CloudFront Functions to normalize cache keys (lowercase, sort query strings)
- Implement cache key normalization for marketing parameters

## Common Issues

- symptoms: "Same page served with different content for different users"
  diagnosis: "Cookie or header that varies per user is not in the cache key."
  resolution: "Add the varying cookie/header to the cache key, or separate into different behaviors."

- symptoms: "Cache hit ratio drops after adding query string forwarding"
  diagnosis: "All query strings forwarded to cache key, including tracking parameters."
  resolution: "Use allowlist to include only functional query strings. Exclude utm_*, fbclid, etc."

- symptoms: "Compressed and uncompressed versions mixed up"
  diagnosis: "Accept-Encoding not in cache key but compression is enabled."
  resolution: "Use CachingOptimized managed policy or add Accept-Encoding to cache key."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
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
3. If edge function was changed, update the distribution to use the previous function version/ARN
4. If access restriction settings were changed, restore original trusted key groups or signers
5. If origin configuration was changed, restore original origin settings including timeouts and protocols
6. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "cache_key — <specific_cause>"
evidence:
  - type: cache_policy
    content: "<cache key components>"
  - type: origin_request_policy
    content: "<forwarded values>"
  - type: cache_hit_ratio
    content: "<before and after metrics>"
severity: MEDIUM
mitigation:
  immediate: "Adjust cache key components"
  long_term: "Implement cache key normalization and separate behaviors for static/dynamic"
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
