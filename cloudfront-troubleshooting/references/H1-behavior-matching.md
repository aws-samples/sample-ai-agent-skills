---
title: "H1 — Cache Behavior Matching Issues"
description: "Diagnose requests matching the wrong cache behavior or path pattern"
status: active
severity: HIGH
triggers:
  - "behavior matching"
  - "wrong behavior"
  - "path pattern"
  - "wrong origin"
  - "request routing"
owner: devops-agent
objective: "Ensure requests match the correct cache behavior and route to the correct origin"
context: "CloudFront evaluates cache behaviors in order — first match wins (NOT longest match). The default behavior (*) matches everything not matched by other behaviors. Path patterns are case-sensitive and support wildcards (* and ?). Common issues include behavior ordering, overlapping patterns, and case sensitivity."
---

## Phase 1 — Triage

MUST:
- List all behaviors and their order: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.CacheBehaviors.Items[*].{Path:PathPattern,Origin:TargetOriginId}'`
- Check default behavior: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.TargetOriginId'`
- Identify which behavior the request matches (test with the actual URL path)
- Verify path patterns are case-sensitive matches

SHOULD:
- Check for overlapping patterns where order matters
- Verify each behavior routes to the correct origin
- Test with curl and check X-Amz-Cf-Pop and Via headers

MAY:
- Check CloudFront access logs for behavior matching patterns
- Map out all behaviors and their origins visually

## Phase 2 — Remediate

MUST:
- Order behaviors from most specific to least specific
- Ensure path patterns match the actual URL paths (case-sensitive)
- Verify each behavior's origin, cache policy, and function associations

SHOULD:
- Use specific path patterns (/api/v1/*) before general ones (/api/*)
- Document behavior ordering and purpose
- Test all path patterns after changes

MAY:
- Use CloudFront Functions to normalize URL paths before behavior matching
- Implement infrastructure as code for consistent behavior management

## Common Issues

- symptoms: "API requests going to static content origin"
  diagnosis: "/api/* behavior is after a more general /* behavior that matches first."
  resolution: "Reorder behaviors — put /api/* before more general patterns."

- symptoms: "Case-sensitive path not matching"
  diagnosis: "Path pattern /Images/* does not match /images/photo.jpg."
  resolution: "Use CloudFront Functions to normalize URL case, or add patterns for both cases."

- symptoms: "Wildcard pattern too broad"
  diagnosis: "/*.jpg matches /api/data.jpg unintentionally."
  resolution: "Use more specific patterns like /images/*.jpg."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "behavior_matching — <specific_cause>"
evidence:
  - type: behavior_order
    content: "<ordered list of behaviors and patterns>"
  - type: matched_behavior
    content: "<which behavior the request matched>"
  - type: expected_behavior
    content: "<which behavior should have matched>"
severity: HIGH
mitigation:
  immediate: "Reorder or fix behavior path patterns"
  long_term: "Document behavior ordering and implement IaC"
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
