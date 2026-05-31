---
title: "B4 — Origin Failover Issues"
description: "Diagnose origin group failover not triggering or misconfigured"
status: active
severity: HIGH
triggers:
  - "origin failover"
  - "origin group"
  - "secondary origin"
  - "failover not working"
owner: devops-agent
objective: "Ensure origin failover works correctly for high availability"
context: "CloudFront origin groups contain a primary and secondary origin. Failover triggers when the primary returns configured HTTP status codes (500, 502, 503, 504) or when CloudFront cannot connect. 4xx errors do NOT trigger failover. Failover is per-request, not a permanent switch."
---

## Phase 1 — Triage

MUST:
- Check origin group configuration: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.OriginGroups'`
- Verify failover criteria (which status codes trigger failover)
- Check both primary and secondary origin health
- Check CloudWatch origin error metrics for both origins

SHOULD:
- Verify the cache behavior references the origin group ID, not an individual origin
- Test failover by simulating primary origin failure
- Check that secondary origin has the same content/configuration as primary

MAY:
- Check CloudFront access logs for failover events (x-edge-detailed-result-type)
- Monitor failover frequency in CloudWatch

## Phase 2 — Remediate

MUST:
- Ensure the cache behavior uses the origin group ID as its origin
- Configure appropriate failover status codes (500, 502, 503, 504)
- Verify both origins are accessible and serve the same content

SHOULD:
- Test failover regularly
- Ensure secondary origin can handle full traffic load
- Use different regions or providers for primary and secondary origins

MAY:
- Combine with Route 53 health checks for DNS-level failover
- Implement Origin Shield to reduce failover impact

## Common Issues

- symptoms: "Failover not triggering on 403/404 errors"
  diagnosis: "Origin failover only triggers on 5xx errors and connection failures, not 4xx."
  resolution: "Use custom error pages for 4xx handling. Failover is not designed for 4xx."

- symptoms: "Cache behavior not using origin group"
  diagnosis: "Behavior references individual origin ID instead of origin group ID."
  resolution: "Update cache behavior to reference the origin group ID."

- symptoms: "Failover triggers but secondary also fails"
  diagnosis: "Secondary origin has the same issue as primary (e.g., shared backend)."
  resolution: "Use truly independent secondary origin (different region, different infrastructure)."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
3. If origin configuration was changed, restore original origin settings including timeouts and protocols
4. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "origin_failover — <specific_cause>"
evidence:
  - type: origin_group_config
    content: "<origin group and failover criteria>"
  - type: behavior_config
    content: "<cache behavior origin reference>"
  - type: origin_health
    content: "<primary and secondary origin status>"
severity: HIGH
mitigation:
  immediate: "Fix origin group configuration or origin health"
  long_term: "Implement independent secondary origin and regular failover testing"
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
