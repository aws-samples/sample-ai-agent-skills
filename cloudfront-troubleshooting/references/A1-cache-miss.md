---
title: "A1 — CloudFront Cache Miss / Low Cache Hit Ratio"
description: "Diagnose why CloudFront is not caching content or has a low cache hit ratio"
status: active
severity: HIGH
triggers:
  - "cache miss"
  - "low cache hit ratio"
  - "not caching"
  - "Miss from cloudfront"
  - "X-Cache: Miss"
owner: devops-agent
objective: "Identify why content is not being cached at CloudFront edge locations and improve cache hit ratio"
context: "CloudFront caches content based on cache key components (URL, headers, cookies, query strings). A low cache hit ratio means most requests go to the origin, increasing latency and origin load. Common causes include overly broad cache keys, Cache-Control: no-cache from the origin, or misconfigured cache policies."
---

## Phase 1 — Triage

MUST:
- Check cache hit ratio in CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name CacheHitRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 3600 --statistics Average`
- Inspect response headers for cache status: `curl -sI https://<domain>/<path> | grep -i 'x-cache\|age\|cache-control'`
- Get the cache policy for the matching behavior: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.CachePolicyId'`
- Check cache policy details: `aws cloudfront get-cache-policy --id <cache-policy-id>`
- Check origin response headers for Cache-Control/Expires directives

SHOULD:
- Check if query strings, headers, or cookies are included in the cache key unnecessarily
- Verify the origin request policy is not forwarding extra headers that fragment the cache
- Check if Origin Shield is enabled to consolidate cache: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].OriginShield'`

MAY:
- Enable CloudFront access logs to analyze cache hit/miss patterns: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Logging'`
- Check if the content varies by Accept-Encoding and compression is configured

## Phase 2 — Remediate

MUST:
- Minimize cache key components — only include headers, cookies, and query strings that actually vary the response
- Ensure the origin sends appropriate Cache-Control headers (e.g., Cache-Control: public, max-age=86400)
- Set appropriate MinTTL in the cache policy to prevent origin no-cache from bypassing CloudFront cache

SHOULD:
- Use managed cache policies (e.g., CachingOptimized) where possible
- Enable Origin Shield for origins with viewers in multiple regions
- Separate static and dynamic content into different cache behaviors

MAY:
- Use versioned file names (style.v2.css) instead of relying on invalidations
- Enable real-time logs for detailed cache analysis

## Common Issues

- symptoms: "X-Cache: Miss from cloudfront on every request"
  diagnosis: "Origin sends Cache-Control: no-store or no-cache and MinTTL is 0."
  resolution: "Set MinTTL > 0 in the cache policy, or fix origin Cache-Control headers."

- symptoms: "Cache hit ratio is low despite cacheable content"
  diagnosis: "Cache key includes unnecessary query strings or headers, fragmenting the cache."
  resolution: "Reduce cache key to only essential components. Use allowlist instead of forwarding all."

- symptoms: "Same content cached multiple times with different keys"
  diagnosis: "Query string parameters like utm_source are in the cache key."
  resolution: "Exclude marketing/tracking query strings from the cache key."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
root_cause: "cache_miss — <specific_cause>"
evidence:
  - type: cache_hit_ratio
    content: "<CloudWatch metric value>"
  - type: cache_policy
    content: "<cache key configuration>"
  - type: origin_headers
    content: "<Cache-Control header from origin>"
severity: HIGH
mitigation:
  immediate: "Adjust cache key or origin headers to improve caching"
  long_term: "Implement cache policy best practices and Origin Shield"
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
