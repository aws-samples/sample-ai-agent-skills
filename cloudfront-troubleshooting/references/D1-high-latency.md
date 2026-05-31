---
title: "D1 — High Latency"
description: "Diagnose high latency in CloudFront content delivery"
status: active
severity: HIGH
triggers:
  - "high latency"
  - "slow loading"
  - "slow response"
  - "TTFB"
  - "time to first byte"
owner: devops-agent
objective: "Identify and reduce latency in CloudFront content delivery"
context: "CloudFront latency has multiple components: DNS resolution, TCP/TLS handshake, edge processing, cache lookup, origin fetch (on miss), and data transfer. High latency can be caused by low cache hit ratio, slow origin, no compression, suboptimal edge location, or missing Origin Shield."
---

## Phase 1 — Triage

MUST:
- Check origin latency: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name OriginLatency --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Check cache hit ratio: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name CacheHitRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 3600 --statistics Average`
- Measure TTFB: `curl -o /dev/null -s -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://<domain>/<path>`
- Check if compression is enabled and working

SHOULD:
- Check if Origin Shield is enabled: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].OriginShield'`
- Check if HTTP/2 is enabled: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.HttpVersion'`
- Verify edge location serving the request (check X-Amz-Cf-Pop response header)

MAY:
- Enable additional CloudWatch metrics for detailed latency breakdown
- Check if HTTP/3 (QUIC) would benefit the use case
- Analyze access logs for latency patterns by edge location

## Phase 2 — Remediate

MUST:
- Improve cache hit ratio (see A1-cache-miss runbook)
- Enable compression for compressible content types
- Optimize origin response time if origin latency is high

SHOULD:
- Enable Origin Shield in the region closest to the origin
- Enable HTTP/2 (default) and consider HTTP/3 for mobile/lossy networks
- Use appropriate TTLs to maximize caching

MAY:
- Use Lambda@Edge to generate responses at the edge for dynamic content
- Implement connection prewarming for critical paths
- Consider regional edge caches for long-tail content

## Common Issues

- symptoms: "High TTFB on first request, fast on subsequent"
  diagnosis: "Cache miss on first request — content fetched from origin."
  resolution: "Expected behavior. Improve cache hit ratio and enable Origin Shield."

- symptoms: "Consistently high latency even on cache hits"
  diagnosis: "Large uncompressed responses or TLS negotiation overhead."
  resolution: "Enable compression. Ensure HTTP/2 is enabled. Check TLS session resumption."

- symptoms: "Latency spikes at specific times"
  diagnosis: "Origin overload during peak traffic causing slow responses."
  resolution: "Scale origin, increase cache TTLs, enable Origin Shield to reduce origin load."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Network configuration changes: YELLOW - May affect connectivity"
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
root_cause: "high_latency — <specific_cause>"
evidence:
  - type: origin_latency
    content: "<origin latency metrics>"
  - type: cache_hit_ratio
    content: "<cache hit ratio>"
  - type: ttfb_breakdown
    content: "<DNS, connect, TLS, TTFB times>"
severity: HIGH
mitigation:
  immediate: "Address primary latency contributor"
  long_term: "Implement Origin Shield, compression, and cache optimization"
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
