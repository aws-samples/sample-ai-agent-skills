---
title: "F2 — CloudFront Functions Errors"
description: "Diagnose CloudFront Functions execution errors and limitations"
status: active
severity: HIGH
triggers:
  - "CloudFront Function"
  - "function error"
  - "function timeout"
  - "function execution"
  - "CF function"
owner: devops-agent
objective: "Resolve CloudFront Functions execution errors and configuration issues"
context: "CloudFront Functions are lightweight JavaScript functions for viewer request/response events. They have strict limits: 10 ms execution, 2 MB function size, 10 KB response body. No network access, no file system. They run on every request with sub-millisecond startup. Use for URL rewrites, header manipulation, redirects, and cache key normalization."
---

## Phase 1 — Triage

MUST:
- Check function association: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.FunctionAssociations'`
- Get function details: `aws cloudfront describe-function --name <function-name>`
- Get function code: `aws cloudfront get-function --name <function-name>`
- Check function metrics: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionExecutionErrors --dimensions Name=DistributionId,Value=<dist-id> Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Test function with test events: `aws cloudfront test-function --name <function-name> --if-match <etag> --event-object <base64-event> --stage DEVELOPMENT`
- Check if function is in LIVE stage (DEVELOPMENT stage is not active)
- Verify function handles all expected input patterns

MAY:
- Check function compute utilization metric (percentage of 10 ms limit used)
- Review function code for common JavaScript errors

## Phase 2 — Remediate

MUST:
- Ensure function is published to LIVE stage
- Keep execution under 10 ms — avoid complex computations
- Return valid CloudFront event objects
- Keep function size under 2 MB

SHOULD:
- Test thoroughly with test-function API before publishing
- Use simple string operations instead of regex where possible
- Handle edge cases (missing headers, unexpected input)

MAY:
- Monitor FunctionComputeUtilization to detect functions approaching the 10 ms limit
- Implement A/B testing by associating different functions with different behaviors

## Common Issues

- symptoms: "Function execution error on every request"
  diagnosis: "JavaScript syntax error or runtime exception in function code."
  resolution: "Test with test-function API. Fix syntax/runtime errors."

- symptoms: "Function works in test but not in production"
  diagnosis: "Function is in DEVELOPMENT stage, not published to LIVE."
  resolution: "Publish function to LIVE stage: aws cloudfront publish-function --name <name> --if-match <etag>"

- symptoms: "Function timeout (10 ms exceeded)"
  diagnosis: "Complex regex or large data processing exceeding 10 ms limit."
  resolution: "Simplify logic. Use Lambda@Edge for complex processing."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Network configuration changes: YELLOW - May affect connectivity"
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
3. If edge function was changed, update the distribution to use the previous function version/ARN
4. If access restriction settings were changed, restore original trusted key groups or signers
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "cloudfront_function — <specific_cause>"
evidence:
  - type: function_config
    content: "<function name, stage, associations>"
  - type: function_errors
    content: "<execution error metrics>"
  - type: test_result
    content: "<test-function output>"
severity: HIGH
mitigation:
  immediate: "Fix function code or publish to LIVE stage"
  long_term: "Implement testing pipeline and compute utilization monitoring"
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
