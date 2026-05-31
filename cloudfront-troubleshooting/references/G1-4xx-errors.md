---
title: "G1 — 4xx Client Errors"
description: "Diagnose 4xx error responses from CloudFront"
status: active
severity: HIGH
triggers:
  - "4xx error"
  - "403 Forbidden"
  - "404 Not Found"
  - "405 Method Not Allowed"
  - "414 URI Too Long"
owner: devops-agent
objective: "Identify the source and cause of 4xx errors from CloudFront"
context: "4xx errors from CloudFront can originate from CloudFront itself (e.g., geo-restriction, signed URL failure, method not allowed) or be passed through from the origin. Check the X-Cache header and error page to determine the source. Common CloudFront-generated 4xx: 403 (geo-restriction, signed URL, WAF), 405 (method not allowed), 414 (URI too long, max 8,192 bytes)."
---

## Phase 1 — Triage

MUST:
- Check 4xx error rate: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 4xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check specific error codes: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 403ErrorRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Test the failing request: `curl -sI https://<domain>/<path>`
- Determine if error is from CloudFront or origin (check error page content and headers)

SHOULD:
- Check geo-restriction: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Restrictions'`
- Check WAF association: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.WebACLId'`
- Check allowed HTTP methods on the matching behavior
- Check if signed URLs/cookies are required

MAY:
- Check CloudFront access logs for detailed error information
- Check WAF logs if WAF is associated: `aws wafv2 get-web-acl --id <acl-id> --scope CLOUDFRONT --region us-east-1`

## Phase 2 — Remediate

MUST:
- For 403 from geo-restriction: adjust geo-restriction settings
- For 403 from signed URL: fix signing configuration (see E1)
- For 403 from WAF: review WAF rules
- For 405: add the HTTP method to the cache behavior's allowed methods
- For 404 from origin: fix origin content or path mapping

SHOULD:
- Configure custom error pages for better user experience
- Set appropriate error caching TTL
- Distinguish between CloudFront-generated and origin-generated errors

MAY:
- Implement custom error responses with different HTTP status codes
- Use Lambda@Edge to handle specific error scenarios

## Common Issues

- symptoms: "403 Forbidden on all requests"
  diagnosis: "WAF blocking requests, geo-restriction, or signed URL required."
  resolution: "Check WAF rules, geo-restriction config, and trusted key groups."

- symptoms: "405 Method Not Allowed on POST requests"
  diagnosis: "Cache behavior only allows GET/HEAD methods."
  resolution: "Update behavior to allow GET/HEAD/OPTIONS/PUT/POST/PATCH/DELETE."

- symptoms: "404 on valid paths"
  diagnosis: "Request matches wrong cache behavior routing to wrong origin."
  resolution: "Check behavior path patterns and origin mappings."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
root_cause: "4xx_error — <specific_error_code>: <cause>"
evidence:
  - type: error_rate
    content: "<4xx error rate metrics>"
  - type: error_source
    content: "<CloudFront or origin>"
  - type: configuration
    content: "<relevant config causing the error>"
severity: HIGH
mitigation:
  immediate: "Fix the specific 4xx error cause"
  long_term: "Implement custom error pages and monitoring"
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
