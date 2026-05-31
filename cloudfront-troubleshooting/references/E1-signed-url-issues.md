---
title: "E1 — Signed URL / Signed Cookie Issues"
description: "Diagnose signed URL and signed cookie authentication failures"
status: active
severity: HIGH
triggers:
  - "signed URL"
  - "signed cookie"
  - "private content"
  - "MissingKey"
  - "AccessDenied signed"
owner: devops-agent
objective: "Resolve signed URL and signed cookie authentication issues for private content"
context: "CloudFront signed URLs and cookies restrict access to private content. They use RSA key pairs via trusted key groups (recommended) or CloudFront key pairs (legacy). Signed URLs are for individual files; signed cookies are for multiple files. Common issues include expired signatures, wrong key pair, clock skew, and policy mismatches."
---

## Phase 1 — Triage

MUST:
- Check if the behavior requires signed URLs/cookies: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.TrustedKeyGroups'`
- Check trusted key groups: `aws cloudfront get-key-group --id <key-group-id>`
- Check public key: `aws cloudfront get-public-key --id <public-key-id>`
- Verify the signed URL/cookie has not expired
- Check the error response (403 AccessDenied with specific CloudFront error codes)

SHOULD:
- Verify the signing key matches a key in the trusted key group
- Check for clock skew between the signing server and CloudFront
- Verify the policy (canned or custom) matches the requested URL

MAY:
- Decode the signed URL policy to verify its contents
- Check CloudFront access logs for signed URL rejection reasons

## Phase 2 — Remediate

MUST:
- Use trusted key groups (not legacy CloudFront key pairs)
- Ensure the public key in the trusted key group matches the private key used for signing
- Set appropriate expiry times accounting for clock skew

SHOULD:
- Use custom policies for IP restrictions and date ranges
- Rotate signing keys regularly using key group updates
- Use signed cookies for multi-file access (e.g., HLS streaming)

MAY:
- Implement key rotation automation
- Use Lambda@Edge for dynamic signed URL generation

## Common Issues

- symptoms: "403 AccessDenied on signed URL"
  diagnosis: "Signed URL expired or signing key not in trusted key group."
  resolution: "Verify expiry time and ensure signing key matches trusted key group."

- symptoms: "Signed cookies not working for subdirectory content"
  diagnosis: "Cookie domain or path does not match the request."
  resolution: "Set cookie Domain to the CloudFront domain and Path to / or the appropriate prefix."

- symptoms: "Signed URL works for one file but not another"
  diagnosis: "Canned policy is URL-specific. Custom policy needed for wildcards."
  resolution: "Use custom policy with wildcard resource pattern."


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
4. If access restriction settings were changed, restore original trusted key groups or signers
5. If geo-restriction or WAF settings were changed, restore original restriction configuration
6. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "signed_url — <specific_cause>"
evidence:
  - type: trusted_key_groups
    content: "<key group configuration>"
  - type: signed_url_policy
    content: "<policy details>"
  - type: error_response
    content: "<CloudFront error details>"
severity: HIGH
mitigation:
  immediate: "Fix signing configuration or key group"
  long_term: "Implement key rotation and use trusted key groups"
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
