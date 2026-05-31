---
title: "C2 — SNI-Related Issues"
description: "Diagnose Server Name Indication (SNI) issues with CloudFront"
status: active
severity: MEDIUM
triggers:
  - "SNI"
  - "Server Name Indication"
  - "dedicated IP"
  - "old browser"
  - "SSL handshake"
owner: devops-agent
objective: "Resolve SNI-related SSL/TLS issues for CloudFront distributions"
context: "CloudFront supports SNI (free, recommended) and dedicated IP ($600/month per distribution) for custom SSL. SNI requires clients to support TLS SNI extension. Very old clients (Android < 4.0, IE on Windows XP) do not support SNI. Most modern clients support SNI."
---

## Phase 1 — Triage

MUST:
- Check SSL support method: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.ViewerCertificate.SSLSupportMethod'`
- Check minimum protocol version: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.ViewerCertificate.MinimumProtocolVersion'`
- Test SNI connection: `openssl s_client -connect <domain>:443 -servername <domain>`
- Test without SNI: `openssl s_client -connect <domain>:443`

SHOULD:
- Check client distribution — identify if affected clients are legacy
- Verify the security policy matches client requirements (TLSv1.2_2021 recommended)

MAY:
- Check CloudFront access logs for TLS version and cipher suite usage
- Analyze client User-Agent strings to identify legacy clients

## Phase 2 — Remediate

MUST:
- Use sni-only (default, free) unless legacy client support is required
- Set minimum protocol version to TLSv1.2_2021 for best security

SHOULD:
- If legacy clients are required, evaluate the cost of dedicated IP ($600/month)
- Communicate to users that legacy clients should be upgraded

MAY:
- Use CloudFront real-time logs to monitor TLS version distribution
- Set up a migration plan for legacy clients

## Common Issues

- symptoms: "SSL error from very old clients but works in modern browsers"
  diagnosis: "Distribution uses SNI and client does not support SNI extension."
  resolution: "Upgrade clients or switch to dedicated IP SSL ($600/month)."

- symptoms: "Wrong certificate returned without SNI"
  diagnosis: "Without SNI, CloudFront cannot determine which certificate to present."
  resolution: "Ensure clients send SNI or use dedicated IP SSL."

- symptoms: "TLS handshake failure with specific clients"
  diagnosis: "Minimum protocol version is too high for the client."
  resolution: "Lower minimum protocol version if needed, or upgrade clients."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
2. If SSL/TLS settings were changed, restore original certificate and security policy configuration
3. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "sni — <specific_cause>"
evidence:
  - type: ssl_config
    content: "<SSL support method and protocol version>"
  - type: client_info
    content: "<affected client details>"
severity: MEDIUM
mitigation:
  immediate: "Adjust SSL support method or protocol version"
  long_term: "Migrate legacy clients and use TLSv1.2_2021 minimum"
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
