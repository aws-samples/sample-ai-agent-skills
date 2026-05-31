---
title: "H2 — Redirect Loops"
description: "Diagnose infinite redirect loops involving CloudFront"
status: active
severity: HIGH
triggers:
  - "redirect loop"
  - "ERR_TOO_MANY_REDIRECTS"
  - "301 loop"
  - "302 loop"
  - "infinite redirect"
owner: devops-agent
objective: "Break redirect loops between CloudFront, origin, and DNS"
context: "Redirect loops commonly occur when: (1) CloudFront viewer protocol is HTTP→HTTPS but origin redirects HTTPS→HTTP, (2) origin redirects to the CloudFront domain which comes back to the origin, (3) S3 website redirect rules create loops, (4) Lambda@Edge/CloudFront Functions create circular redirects. The most common cause is HTTP/HTTPS protocol mismatch."
---

## Phase 1 — Triage

MUST:
- Follow the redirect chain: `curl -sIL https://<domain>/<path> 2>&1 | grep -i 'location\|HTTP/'`
- Check viewer protocol policy: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.ViewerProtocolPolicy'`
- Check origin protocol policy: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].CustomOriginConfig.OriginProtocolPolicy'`
- Test origin directly to see if it redirects: `curl -sI http://<origin-domain>/<path>`

SHOULD:
- Check if origin redirects to the CloudFront domain (creating a loop)
- Check Lambda@Edge or CloudFront Functions for redirect logic
- Verify Host header forwarding — origin may redirect based on hostname

MAY:
- Check S3 website redirect rules if using S3 website origin
- Check ALB listener rules for redirect configurations

## Phase 2 — Remediate

MUST:
- For HTTP/HTTPS loops: set origin protocol to HTTPS-only if origin supports it, or HTTP-only if origin handles SSL termination
- Ensure origin does not redirect back to the CloudFront domain
- Fix circular redirect logic in edge functions

SHOULD:
- Use "Redirect HTTP to HTTPS" viewer protocol policy with HTTPS-only origin protocol
- Configure origin to not redirect when receiving requests from CloudFront
- Use origin custom headers to identify CloudFront requests at the origin

MAY:
- Implement redirect logic in CloudFront Functions instead of at the origin
- Use separate behaviors for HTTP and HTTPS handling

## Common Issues

- symptoms: "ERR_TOO_MANY_REDIRECTS with HTTPS"
  diagnosis: "Viewer protocol redirects HTTP→HTTPS, but origin protocol is Match Viewer, and origin redirects HTTP→HTTPS again."
  resolution: "Set origin protocol to HTTPS-only, or set to HTTP-only if origin handles SSL termination."

- symptoms: "Redirect loop between www and non-www"
  diagnosis: "CloudFront redirects to www, origin redirects back to non-www."
  resolution: "Handle www/non-www redirect in one place only (CloudFront Function or origin, not both)."

- symptoms: "S3 website origin redirect loop"
  diagnosis: "S3 redirect rule redirects to CloudFront domain, which routes back to S3."
  resolution: "Fix S3 redirect rules to not redirect CloudFront requests."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
root_cause: "redirect_loop — <specific_cause>"
evidence:
  - type: redirect_chain
    content: "<full redirect chain>"
  - type: viewer_protocol
    content: "<viewer protocol policy>"
  - type: origin_protocol
    content: "<origin protocol policy>"
severity: HIGH
mitigation:
  immediate: "Break the redirect loop by fixing protocol or redirect configuration"
  long_term: "Consolidate redirect logic in one layer"
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
