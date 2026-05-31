---
title: "G2 — 5xx Server Errors"
description: "Diagnose 5xx error responses from CloudFront"
status: active
severity: CRITICAL
triggers:
  - "5xx error"
  - "502 Bad Gateway"
  - "503 Service Unavailable"
  - "504 Gateway Timeout"
  - "500 Internal Server Error"
owner: devops-agent
objective: "Identify and resolve 5xx errors from CloudFront"
context: "5xx errors indicate server-side failures. CloudFront generates 502 (origin connection/SSL failure, Lambda@Edge error), 503 (capacity, Lambda@Edge throttle), and 504 (origin timeout). Origin-generated 5xx errors are passed through. Check X-Cache and x-amz-cf-pop headers to identify the source."
---

## Phase 1 — Triage

MUST:
- Check 5xx error rate: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 5xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check origin latency: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name OriginLatency --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Test origin directly: `curl -sI https://<origin-domain>/<path>`
- Check for Lambda@Edge errors if functions are associated
- Check distribution config for origin settings: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins'`

SHOULD:
- Check origin health (ALB health checks, EC2 status)
- Verify origin SSL certificate if using HTTPS
- Check origin timeout settings (connection timeout, read timeout)
- Check if origin failover is configured and working

MAY:
- Check CloudFront access logs for x-edge-detailed-result-type
- Check AWS Health Dashboard for CloudFront service issues
- Check origin server logs for corresponding errors

## Phase 2 — Remediate

MUST:
- For 502 (origin connection): fix origin connectivity (see B1)
- For 502 (SSL): fix origin SSL certificate (see C3)
- For 502 (Lambda@Edge): fix function errors (see F1)
- For 503 (throttle): request limit increase or reduce traffic
- For 504 (timeout): increase origin read timeout or optimize origin response time

SHOULD:
- Configure origin failover for high availability
- Set appropriate timeout values
- Implement custom error pages for 5xx errors

MAY:
- Enable Origin Shield to reduce origin load
- Implement circuit breaker patterns at the origin

## Common Issues

- symptoms: "502 Bad Gateway intermittently"
  diagnosis: "Origin keep-alive timeout < CloudFront's 5s, causing stale connection reuse."
  resolution: "Set origin keep-alive timeout > 5 seconds."

- symptoms: "504 Gateway Timeout on dynamic content"
  diagnosis: "Origin processing time exceeds CloudFront read timeout (default 30s)."
  resolution: "Increase read timeout (max 60s) or optimize origin response time."

- symptoms: "503 Service Unavailable during traffic spikes"
  diagnosis: "Lambda@Edge concurrent execution limit or CloudFront capacity."
  resolution: "Request Lambda@Edge limit increase. Consider CloudFront Functions for simpler logic."


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
4. If edge function was changed, update the distribution to use the previous function version/ARN
5. If origin configuration was changed, restore original origin settings including timeouts and protocols
6. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "5xx_error — <specific_error_code>: <cause>"
evidence:
  - type: error_rate
    content: "<5xx error rate metrics>"
  - type: origin_latency
    content: "<origin latency metrics>"
  - type: origin_health
    content: "<origin status>"
severity: CRITICAL
mitigation:
  immediate: "Fix origin or function error"
  long_term: "Implement origin failover, monitoring, and auto-scaling"
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
