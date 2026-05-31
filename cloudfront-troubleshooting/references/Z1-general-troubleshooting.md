---
title: "Z1 — General CloudFront Troubleshooting (Catch-All)"
description: "Fallback SOP for CloudFront issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown CloudFront issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get distribution overview: `aws cloudfront get-distribution --id <dist-id>`
- Check distribution status: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.Status'`
- Check error rates: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 4xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check 5xx errors: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 5xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check cache hit ratio: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name CacheHitRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 3600 --statistics Average`
- Test basic connectivity: `curl -sI https://<domain>/`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudfront.amazonaws.com --max-results 10`
- Check origins: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins'`
- Check behaviors: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.CacheBehaviors'`
- Check SSL certificate: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.ViewerCertificate'`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Cache miss / stale content → Cache (A1-A4)
- Origin connection / access errors → Origin (B1-B4)
- SSL/TLS errors → SSL/TLS (C1-C3)
- High latency / slow delivery → Performance (D1-D3)
- Signed URL / geo-restriction → Security (E1-E3)
- Edge function errors → Edge Functions (F1-F3)
- 4xx / 5xx errors → Errors (G1-G3)
- Wrong behavior / redirect loops → Routing (H1-H2)
- Missing logs → Logging (I1-I2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all distribution configurations systematically
- Review CloudTrail for recent configuration changes
- Check CloudFront access logs if enabled
- Verify DNS resolution: `nslookup <domain>`

SHOULD:
- Check AWS Health Dashboard for CloudFront service events
- Compare with a known-good distribution configuration
- Check WAF if associated: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.WebACLId'`

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps


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
5. If access restriction settings were changed, restore original trusted key groups or signers
6. If origin configuration was changed, restore original origin settings including timeouts and protocols
7. If geo-restriction or WAF settings were changed, restore original restriction configuration
8. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<cache|origin|ssl|performance|security|edge_functions|errors|routing|logging|unknown>"
investigation_path: "distribution config → CloudWatch → CloudTrail → <domain_classification>"
evidence:
  - type: distribution_config
    content: "<distribution configuration summary>"
  - type: cloudwatch
    content: "<error rates and metrics>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
