---
title: "E2 — Geo-Restriction Issues"
description: "Diagnose geo-restriction (geoblocking) configuration issues"
status: active
severity: MEDIUM
triggers:
  - "geo-restriction"
  - "geoblocking"
  - "country block"
  - "geographic restriction"
  - "403 geo"
owner: devops-agent
objective: "Resolve geo-restriction configuration issues"
context: "CloudFront geo-restriction uses a third-party GeoIP database to allow or deny access based on viewer country. It applies to the entire distribution, not per behavior. It uses ISO 3166-1 alpha-2 country codes. GeoIP is not 100% accurate — VPN and proxy users may bypass it. For per-path restrictions, use Lambda@Edge."
---

## Phase 1 — Triage

MUST:
- Check geo-restriction config: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Restrictions.GeoRestriction'`
- Verify restriction type (whitelist or blacklist) and country codes
- Test from affected region or use a VPN to simulate
- Check if the 403 error is from geo-restriction (check CloudFront error page)

SHOULD:
- Verify country codes are correct ISO 3166-1 alpha-2 format
- Check if WAF geo-match rules are also in effect (separate from CloudFront geo-restriction)
- Verify the viewer's actual country vs expected country

MAY:
- Check CloudFront access logs for blocked requests by country
- Test with known IP addresses from specific countries

## Phase 2 — Remediate

MUST:
- Use correct ISO 3166-1 alpha-2 country codes (US, GB, DE, not USA, UK, GER)
- Choose whitelist (allow only listed) or blacklist (block listed) appropriately
- Update the restriction list as needed

SHOULD:
- Use WAF geo-match rules for per-behavior or more granular geo-restriction
- Implement custom error pages for geo-blocked users
- Document the geo-restriction policy

MAY:
- Use Lambda@Edge for per-path geo-restriction
- Implement geo-based content routing instead of blocking

## Common Issues

- symptoms: "Users in allowed country getting 403"
  diagnosis: "GeoIP database inaccuracy or user is behind a VPN/proxy."
  resolution: "GeoIP is not 100% accurate. Consider using WAF for more control."

- symptoms: "Geo-restriction not blocking expected country"
  diagnosis: "Wrong country code or using blacklist instead of whitelist."
  resolution: "Verify ISO 3166-1 alpha-2 codes and restriction type."

- symptoms: "Need per-path geo-restriction"
  diagnosis: "CloudFront geo-restriction applies to entire distribution."
  resolution: "Use Lambda@Edge or WAF geo-match rules for per-path control."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
2. If edge function was changed, update the distribution to use the previous function version/ARN
3. If access restriction settings were changed, restore original trusted key groups or signers
4. If geo-restriction or WAF settings were changed, restore original restriction configuration
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "geo_restriction — <specific_cause>"
evidence:
  - type: geo_config
    content: "<restriction type and country codes>"
  - type: viewer_location
    content: "<viewer country and IP>"
severity: MEDIUM
mitigation:
  immediate: "Fix geo-restriction configuration"
  long_term: "Implement WAF geo-match for granular control"
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
