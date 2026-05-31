---
title: "A2 — Stale Content Served by CloudFront"
description: "Diagnose why CloudFront serves outdated content after origin updates"
status: active
severity: HIGH
triggers:
  - "stale content"
  - "old version"
  - "outdated content"
  - "content not updating"
  - "cached old file"
owner: devops-agent
objective: "Identify why CloudFront is serving stale content and ensure fresh content is delivered"
context: "CloudFront caches content at edge locations based on TTL settings. When origin content changes, cached copies remain until TTL expires or an invalidation is created. Stale content issues are often caused by long TTLs, missing invalidations, or the TTL hierarchy between origin headers and CloudFront settings."
---

## Phase 1 — Triage

MUST:
- Check the Age header to see how long content has been cached: `curl -sI https://<domain>/<path> | grep -i 'age\|cache-control\|x-cache\|etag\|last-modified'`
- Check the cache policy TTL settings: `aws cloudfront get-cache-policy --id <cache-policy-id>`
- Check origin Cache-Control headers by requesting directly from origin
- Check for pending invalidations: `aws cloudfront list-invalidations --distribution-id <dist-id> --query 'InvalidationList.Items[?Status==`InProgress`]'`

SHOULD:
- Verify the TTL hierarchy: max(MinTTL, min(MaxTTL, origin-header-or-DefaultTTL))
- Check if the origin sends ETag or Last-Modified for conditional requests
- Verify the cache key — different query strings may serve different cached versions

MAY:
- Check CloudFront access logs for cache hit/miss patterns on the specific path
- Test with cache-busting query string: `curl -sI "https://<domain>/<path>?cachebust=$(date +%s)"`

## Phase 2 — Remediate

MUST:
- Create an invalidation if immediate refresh is needed: `aws cloudfront create-invalidation --distribution-id <dist-id> --paths '/<path>'`
- Adjust TTL settings to match content update frequency
- Ensure origin sends appropriate Cache-Control headers

SHOULD:
- Use versioned file names for static assets (main.abc123.js) to avoid invalidation needs
- Set shorter TTLs for frequently changing content
- Configure origin to send ETag/Last-Modified for conditional request support

MAY:
- Implement a deployment pipeline that automatically creates invalidations
- Use wildcard invalidations sparingly (/* counts as one path but invalidates everything)

## Common Issues

- symptoms: "Content updated at origin but CloudFront still serves old version"
  diagnosis: "TTL has not expired and no invalidation was created."
  resolution: "Create an invalidation or wait for TTL expiry. Use versioned filenames."

- symptoms: "Invalidation completed but some users still see old content"
  diagnosis: "Browser cache or intermediate proxy cache is serving stale content."
  resolution: "Check browser cache. Add Cache-Control: no-cache for the specific resource or use versioned URLs."

- symptoms: "MinTTL is 0 but content is still cached for hours"
  diagnosis: "Origin sends Cache-Control: max-age=86400 which overrides when MinTTL is 0."
  resolution: "Fix origin Cache-Control header or set MaxTTL to limit cache duration."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
4. If access restriction settings were changed, restore original trusted key groups or signers
5. If origin configuration was changed, restore original origin settings including timeouts and protocols
6. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "stale_content — <specific_cause>"
evidence:
  - type: age_header
    content: "<Age value from response>"
  - type: cache_policy_ttl
    content: "<MinTTL/DefaultTTL/MaxTTL values>"
  - type: origin_cache_control
    content: "<Cache-Control header from origin>"
severity: HIGH
mitigation:
  immediate: "Create invalidation or adjust TTL"
  long_term: "Implement versioned filenames and appropriate TTL strategy"
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
