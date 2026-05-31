# CloudFront Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CloudFront problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudFront when the console alone isn't enough — cache behavior analysis, origin connection debugging, SSL/TLS certificate issues, edge function errors, performance optimization, security configuration (signed URLs, geo-restriction), routing problems, and logging configuration.

### Activate When

- Cache miss or low cache hit ratio
- Stale content being served after updates
- Invalidation failures or slow propagation
- Origin connection failures (S3 or custom origins)
- S3 origin access denied (OAC/OAI issues)
- Custom origin 502/504 errors
- Origin failover not triggering
- SSL/TLS certificate errors
- SNI-related issues
- Origin SSL handshake failures
- High latency or slow content delivery
- Compression not working
- Signed URL or signed cookie failures
- Geo-restriction not working as expected
- Field-level encryption issues
- Lambda@Edge execution errors
- CloudFront Functions failures
- Function association problems
- 4xx or 5xx error spikes
- Custom error response configuration issues
- Cache behavior path matching problems
- Redirect loops
- Access logs not appearing
- Real-time logs configuration issues

---

## Skill Structure

```
cloudfront-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cache-miss.md
    ├── A2-stale-content.md
    ├── A3-invalidation-failures.md
    ├── A4-cache-key-issues.md
    ├── B1-origin-connection-failures.md
    ├── B2-s3-origin-issues.md
    ├── B3-custom-origin-errors.md
    ├── B4-origin-failover.md
    ├── C1-certificate-errors.md
    ├── C2-sni-issues.md
    ├── C3-origin-ssl.md
    ├── D1-high-latency.md
    ├── D2-bandwidth-throttling.md
    ├── D3-compression.md
    ├── E1-signed-url-issues.md
    ├── E2-geo-restriction.md
    ├── E3-field-level-encryption.md
    ├── F1-lambda-edge-errors.md
    ├── F2-cloudfront-functions.md
    ├── F3-function-association.md
    ├── G1-4xx-errors.md
    ├── G2-5xx-errors.md
    ├── G3-custom-error-responses.md
    ├── H1-behavior-matching.md
    ├── H2-redirect-loops.md
    ├── I1-access-logs.md
    ├── I2-real-time-logs.md
    ├── Z1-general-troubleshooting.md
    ├── cloudfront-guardrails.md
    └── cloudfront-hallucination-patterns.yaml
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cache** | A1–A4 | Cache miss, stale content, invalidation failures, cache key issues |
| **B — Origin** | B1–B4 | Origin connection failures, S3 origin issues, custom origin errors, origin failover |
| **C — SSL/TLS** | C1–C3 | Certificate errors, SNI issues, origin SSL |
| **D — Performance** | D1–D3 | High latency, bandwidth throttling, compression |
| **E — Security** | E1–E3 | Signed URL/cookie issues, geo-restriction, field-level encryption |
| **F — Edge Functions** | F1–F3 | Lambda@Edge errors, CloudFront Functions, function association |
| **G — Errors** | G1–G3 | 4xx errors, 5xx errors, custom error responses |
| **H — Routing** | H1–H2 | Behavior matching, redirect loops |
| **I — Logging** | I1–I2 | Access logs, real-time logs |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/cloudfront-guardrails.md` covering TTL hierarchy, invalidation timing, OAC vs OAI, cache behavior matching, origin failover triggers, Lambda@Edge limits, CloudFront Functions limits, signed URLs vs cookies, custom error page caching, default root object scope, HTTP methods per behavior, and distribution deployment time.

---

## Investigation Workflow

1. **Triage** — Collect distribution config, check CloudWatch error rates and cache metrics
2. **Deep Dive** — Examine cache behaviors, origins, invalidations, certificates
3. **Detailed** — Inspect cache policies, origin request policies, edge functions, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `cloudfront:*`, `acm:DescribeCertificate`, `lambda:GetFunction`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `s3:GetBucketPolicy`, `wafv2:GetWebACL`
- CloudFront access logs enabled (recommended)
- CloudWatch metrics enabled for the distribution

---

## Usage Examples

```
# Get distribution overview
aws cloudfront get-distribution --id E1234567890

# Check cache hit ratio
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront \
  --metric-name CacheHitRate --dimensions Name=DistributionId,Value=E1234567890 Name=Region,Value=Global \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 3600 --statistics Average

# List recent invalidations
aws cloudfront list-invalidations --distribution-id E1234567890

# Check origin errors
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront \
  --metric-name OriginLatency --dimensions Name=DistributionId,Value=E1234567890 Name=Region,Value=Global \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Average
```

---

## License

MIT-0
