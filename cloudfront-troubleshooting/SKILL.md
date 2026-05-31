---
name: cloudfront-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CloudFront problems
  by analyzing distribution configurations, cache behaviors, origin settings,
  SSL/TLS certificates, edge functions, and following structured runbooks.
  Activate when: cache miss issues, stale content, invalidation failures,
  origin connection errors, S3 origin access problems, custom origin errors,
  SSL/TLS certificate issues, high latency, compression problems, signed
  URL/cookie failures, geo-restriction issues, Lambda@Edge errors,
  CloudFront Functions failures, 4xx/5xx errors, redirect loops, behavior
  matching issues, access log problems, or the user says something is wrong
  with CloudFront without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with CloudFront, S3, ACM, IAM, CloudWatch,
  CloudTrail, Lambda, and optionally WAF and Route 53 permissions.
---

# CloudFront Diagnostics

## When to use

Any CloudFront investigation where the console alone is insufficient — cache behavior analysis, origin troubleshooting, SSL/TLS debugging, edge function errors, performance optimization, security configuration, or routing issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudfront get-distribution --id <distribution-id>
aws cloudfront get-distribution-config --id <distribution-id>
aws cloudfront list-distributions --query 'DistributionList.Items[*].{Id:Id,Domain:DomainName,Status:Status}'
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name Requests --dimensions Name=DistributionId,Value=<distribution-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 4xxErrorRate --dimensions Name=DistributionId,Value=<distribution-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 5xxErrorRate --dimensions Name=DistributionId,Value=<distribution-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average
```

### Step 2 — Domain deep dive

```
aws cloudfront get-distribution-config --id <distribution-id> --query 'DistributionConfig.CacheBehaviors'
aws cloudfront get-distribution-config --id <distribution-id> --query 'DistributionConfig.Origins'
aws cloudfront list-invalidations --distribution-id <distribution-id>
aws cloudfront get-invalidation --distribution-id <distribution-id> --id <invalidation-id>
aws acm describe-certificate --certificate-arn <cert-arn> --region us-east-1
aws cloudfront list-cloud-front-origin-access-identities
aws cloudfront list-origin-access-controls
```

### Step 3 — Detailed investigation

```
aws cloudfront get-cache-policy --id <cache-policy-id>
aws cloudfront get-origin-request-policy --id <origin-request-policy-id>
aws cloudfront get-response-headers-policy --id <response-headers-policy-id>
aws lambda get-function --function-name <function-name> --qualifier <version>
aws cloudfront describe-function --name <function-name>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudfront.amazonaws.com
aws cloudfront get-realtime-log-config --name <config-name>
```

Read `references/cloudfront-guardrails.md` before concluding on any CloudFront issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-distribution` | Full distribution details |
| `get-distribution-config` | Distribution configuration |
| `list-invalidations` | Check invalidation status |
| `get-cache-policy` | Cache key and TTL settings |
| `get-origin-request-policy` | Headers/cookies/query strings forwarded to origin |
| `get-response-headers-policy` | Security headers, CORS |
| `list-origin-access-controls` | OAC configuration for S3 |
| `describe-function` | CloudFront Functions details |
| `acm describe-certificate` | SSL/TLS certificate status |
| `get-realtime-log-config` | Real-time log configuration |
| `get-monitoring-subscription` | Additional metrics |
| `list-functions` | List CloudFront Functions |

## Gotchas: CloudFront

- Cache behavior precedence: behaviors are evaluated in order by path pattern. The first match wins. The default (*) behavior is always last. Longest match does NOT win — order matters.
- Origin failover: only triggers on 5xx errors from the origin or connection failures. 4xx errors do NOT trigger failover. Configure an origin group with primary and secondary origins.
- TTL hierarchy: Cache-Control/Expires headers from the origin take precedence over CloudFront cache policy TTL settings, UNLESS the cache policy minimum TTL is greater than the origin header value. Order: max(MinTTL, min(MaxTTL, origin-header-or-DefaultTTL)).
- Invalidation costs: first 1,000 invalidation paths per month are free. After that, $0.005 per path. Invalidations are NOT instant — they propagate to all edge locations and typically take 1-2 minutes but can take up to 10-15 minutes. Use versioned file names instead when possible.
- OAC vs OAI: Origin Access Control (OAC) is the recommended method for S3 origins. OAI (Origin Access Identity) is legacy. OAC supports SSE-KMS, all S3 regions, and S3 Object Lambda. OAI does not support SSE-KMS or newer S3 features.
- Lambda@Edge limits: viewer request/response functions have 5-second timeout and 128 MB memory. Origin request/response functions have 30-second timeout and 10 GB memory (configurable). Functions must be in us-east-1. No environment variables, no VPC, no layers, no ARM.
- CloudFront Functions limits: 10 ms execution time, 2 MB max function size, 10 KB max response body for viewer response events. No network access, no file system access. Only viewer request and viewer response events.
- Custom error pages: cached separately with their own TTL (Error Caching Minimum TTL). Changing the error page requires invalidation of the error response. Can map HTTP error codes to different response codes.
- Geo-restriction: uses a third-party GeoIP database. Not 100% accurate. Applies to the entire distribution, not per behavior. Use signed URLs/cookies or Lambda@Edge for per-path geo-restriction.
- Signed URLs vs signed cookies: signed URLs are for individual files. Signed cookies are for multiple files (e.g., HLS video segments). Both use CloudFront key pairs or trusted key groups. Key pairs are legacy — use trusted key groups.
- Field-level encryption: encrypts specific POST body fields at the edge. Uses RSA encryption with a public key. Only the application with the private key can decrypt. Limited to 10 fields and 1 MB total POST body.
- HTTP/2 is enabled by default. HTTP/3 (QUIC) must be explicitly enabled. Both only apply to viewer-to-CloudFront connections. CloudFront-to-origin always uses HTTP/1.1.
- WebSocket support: requires the distribution to allow GET and HEAD methods (minimum). WebSocket connections upgrade from HTTP. The origin must support WebSocket. Idle timeout is 10 minutes.
- Origin Shield: adds an additional caching layer between edge locations and the origin. Reduces origin load. Choose the Origin Shield region closest to your origin. Adds per-request cost.

## Anti-hallucination rules

1. Always cite specific distribution configurations, cache policies, or API responses as evidence.
2. Cache behavior order matters — first match wins, NOT longest match. Never claim longest-match behavior.
3. Invalidations are NOT instant. Never claim content is immediately updated after invalidation.
4. OAI is legacy. Never recommend OAI for new distributions — always recommend OAC.
5. Lambda@Edge functions MUST be in us-east-1. Never suggest deploying them in other regions.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cache | A1-A4 | Cache miss, stale content, invalidation failures, cache key issues |
| B — Origin | B1-B4 | Origin connection failures, S3 origin issues, custom origin errors, origin failover |
| C — SSL/TLS | C1-C3 | Certificate errors, SNI issues, origin SSL |
| D — Performance | D1-D3 | High latency, bandwidth throttling, compression |
| E — Security | E1-E3 | Signed URL/cookie issues, geo-restriction, field-level encryption |
| F — Edge Functions | F1-F3 | Lambda@Edge errors, CloudFront Functions, function association |
| G — Errors | G1-G3 | 4xx errors, 5xx errors, custom error responses |
| H — Routing | H1-H2 | Behavior matching, redirect loops |
| I — Logging | I1-I2 | Access logs, real-time logs |
| Z — Catch-All | Z1 | General troubleshooting |
