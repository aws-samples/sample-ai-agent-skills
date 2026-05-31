---
title: "A3 — CloudFront Invalidation Failures"
description: "Diagnose invalidation issues including slow propagation, failed invalidations, and cost concerns"
status: active
severity: MEDIUM
triggers:
  - "invalidation failed"
  - "invalidation slow"
  - "invalidation not working"
  - "cache not clearing"
  - "invalidation stuck"
owner: devops-agent
objective: "Resolve invalidation failures and optimize invalidation strategy"
context: "CloudFront invalidations remove cached content from edge locations. They propagate asynchronously and typically take 1-2 minutes but can take up to 10-15 minutes. The first 1,000 paths per month are free; additional paths cost $0.005 each. Wildcard invalidations (/*) count as one path."
---

## Phase 1 — Triage

MUST:
- Check invalidation status: `aws cloudfront get-invalidation --distribution-id <dist-id> --id <invalidation-id>`
- List recent invalidations: `aws cloudfront list-invalidations --distribution-id <dist-id> --max-items 10`
- Verify the invalidation path matches the actual URL path (case-sensitive, must start with /)
- Check distribution status: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.Status'`

SHOULD:
- Verify the path pattern is correct — /images/* invalidates all under /images/, not /images itself
- Check if there are concurrent invalidation limits (3 wildcard invalidations in progress at a time)
- Confirm the content is actually cached (check X-Cache header before invalidating)

MAY:
- Check CloudTrail for invalidation API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateInvalidation`
- Monitor invalidation costs if running many invalidations

## Phase 2 — Remediate

MUST:
- Wait for in-progress invalidations to complete before assuming failure (up to 15 minutes)
- Use correct path format: paths must start with / and are case-sensitive
- For immediate needs, use versioned filenames instead of invalidations

SHOULD:
- Batch invalidation paths into a single request (up to 3,000 paths per invalidation)
- Use wildcard /* only when necessary — it invalidates the entire distribution cache
- Implement versioned filenames in the build pipeline to reduce invalidation dependency

MAY:
- Set up CloudWatch alarms for invalidation completion
- Automate invalidations in CI/CD pipelines with proper wait logic

## Common Issues

- symptoms: "Invalidation shows Completed but content is still stale"
  diagnosis: "Browser cache or CDN proxy between viewer and CloudFront is caching."
  resolution: "Clear browser cache. Check for intermediate proxies. Verify with curl -H 'Cache-Control: no-cache'."

- symptoms: "Invalidation path does not match any cached content"
  diagnosis: "Path is case-sensitive and must match the exact URL path including query strings if they are part of the cache key."
  resolution: "Verify the exact cached path. Use /* for broad invalidation."

- symptoms: "Too many invalidation requests causing costs"
  diagnosis: "Frequent deployments triggering individual file invalidations."
  resolution: "Use versioned filenames. Batch paths into single requests. Use /* for full deployments."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
3. If access restriction settings were changed, restore original trusted key groups or signers
4. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "invalidation_failure — <specific_cause>"
evidence:
  - type: invalidation_status
    content: "<invalidation status and paths>"
  - type: distribution_status
    content: "<distribution deployment status>"
severity: MEDIUM
mitigation:
  immediate: "Fix invalidation path or wait for propagation"
  long_term: "Adopt versioned filenames to eliminate invalidation dependency"
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
