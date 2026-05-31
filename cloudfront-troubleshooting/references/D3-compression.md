---
title: "D3 — Compression Issues"
description: "Diagnose CloudFront compression not working or misconfigured"
status: active
severity: MEDIUM
triggers:
  - "compression"
  - "gzip"
  - "brotli"
  - "Content-Encoding"
  - "not compressed"
owner: devops-agent
objective: "Ensure CloudFront compression is working correctly to reduce transfer sizes"
context: "CloudFront can compress content using gzip and Brotli. Compression must be enabled in the cache behavior AND the cache key must include Accept-Encoding. CloudFront compresses objects between 1,000 bytes and 10 MB. The origin must NOT already compress the content if CloudFront compression is enabled. Content-Type must be a compressible type."
---

## Phase 1 — Triage

MUST:
- Check if compression is enabled: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.Compress'`
- Check cache policy includes Accept-Encoding: `aws cloudfront get-cache-policy --id <cache-policy-id>`
- Test compression: `curl -sI -H "Accept-Encoding: gzip, br" https://<domain>/<path> | grep -i 'content-encoding\|content-length\|content-type'`
- Verify the Content-Type is compressible (text/html, application/javascript, text/css, etc.)

SHOULD:
- Check if origin is already compressing (CloudFront won't double-compress)
- Verify object size is between 1,000 bytes and 10 MB
- Check if the response has Content-Length header (required for compression)

MAY:
- Compare compressed vs uncompressed response sizes
- Check access logs for Content-Encoding patterns

## Phase 2 — Remediate

MUST:
- Enable Compress Objects Automatically in the cache behavior
- Use a cache policy that includes Accept-Encoding (e.g., CachingOptimized)
- Ensure origin does NOT compress if CloudFront compression is enabled

SHOULD:
- Use CachingOptimized managed cache policy (includes Accept-Encoding automatically)
- Verify all compressible content types are served with correct Content-Type
- Remove origin compression if CloudFront compression is preferred

MAY:
- Configure origin to compress for content types CloudFront doesn't handle
- Monitor compression ratio in access logs

## Common Issues

- symptoms: "Content-Encoding header missing despite compression enabled"
  diagnosis: "Cache policy does not include Accept-Encoding in cache key."
  resolution: "Use CachingOptimized policy or add Accept-Encoding to cache key."

- symptoms: "Small files not compressed"
  diagnosis: "Objects under 1,000 bytes are not compressed by CloudFront."
  resolution: "Expected behavior. Only objects 1,000 bytes to 10 MB are compressed."

- symptoms: "Origin already compresses, CloudFront serves double-encoded"
  diagnosis: "Both origin and CloudFront compression enabled."
  resolution: "Disable compression at either origin or CloudFront, not both."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
root_cause: "compression — <specific_cause>"
evidence:
  - type: compression_config
    content: "<compress setting and cache policy>"
  - type: response_headers
    content: "<Content-Encoding and Content-Type>"
  - type: content_size
    content: "<compressed vs uncompressed size>"
severity: MEDIUM
mitigation:
  immediate: "Fix compression configuration"
  long_term: "Use CachingOptimized policy and standardize compression strategy"
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
