---
title: "D2 — Bandwidth Throttling"
description: "Diagnose bandwidth limitations and transfer rate issues"
status: active
severity: MEDIUM
triggers:
  - "bandwidth"
  - "throttling"
  - "slow download"
  - "transfer rate"
  - "rate limit"
owner: devops-agent
objective: "Identify and resolve bandwidth throttling or transfer rate limitations"
context: "CloudFront does not impose per-distribution bandwidth limits by default, but AWS account-level limits exist (default 150 Gbps per distribution, requestable increase). Slow transfers can be caused by origin bandwidth limits, viewer connection speed, TCP window sizing, or large file delivery without range requests."
---

## Phase 1 — Triage

MUST:
- Check total bytes downloaded: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name BytesDownloaded --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check request count for traffic volume: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name Requests --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check origin latency for origin-side bottlenecks
- Verify if the issue is specific to certain regions or global

SHOULD:
- Check AWS Service Quotas for CloudFront limits: `aws service-quotas get-service-quota --service-code cloudfront --quota-code L-5FC1D6E0`
- Check if origin is bandwidth-limited
- Verify large files use range requests (Accept-Ranges header)

MAY:
- Check CloudFront access logs for transfer times by file size
- Monitor origin bandwidth utilization

## Phase 2 — Remediate

MUST:
- Request limit increase if hitting CloudFront account limits
- Optimize origin bandwidth if origin is the bottleneck
- Enable compression to reduce transfer sizes

SHOULD:
- Use range requests for large files (video, downloads)
- Enable Origin Shield to reduce origin bandwidth
- Distribute traffic across multiple origins if needed

MAY:
- Use CloudFront price class to control which edge locations are used
- Implement adaptive bitrate streaming for video content

## Common Issues

- symptoms: "Slow downloads for large files"
  diagnosis: "Large files served without range request support or compression."
  resolution: "Enable range requests at origin. Use multipart downloads."

- symptoms: "Bandwidth drops during peak hours"
  diagnosis: "Origin bandwidth saturated during peak traffic."
  resolution: "Scale origin, enable Origin Shield, increase cache TTLs."

- symptoms: "Transfer rate limited to specific throughput"
  diagnosis: "Hitting CloudFront per-distribution bandwidth limit."
  resolution: "Request limit increase via AWS Support."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "bandwidth — <specific_cause>"
evidence:
  - type: bandwidth_metrics
    content: "<bytes downloaded and request count>"
  - type: service_quotas
    content: "<CloudFront limits>"
severity: MEDIUM
mitigation:
  immediate: "Address bandwidth bottleneck"
  long_term: "Implement compression, range requests, and Origin Shield"
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
