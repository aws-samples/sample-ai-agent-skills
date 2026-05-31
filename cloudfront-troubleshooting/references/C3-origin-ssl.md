---
title: "C3 — Origin SSL/TLS Issues"
description: "Diagnose SSL/TLS handshake failures between CloudFront and the origin"
status: active
severity: HIGH
triggers:
  - "origin SSL"
  - "origin TLS"
  - "origin handshake"
  - "502 SSL"
  - "origin certificate"
owner: devops-agent
objective: "Resolve SSL/TLS issues between CloudFront and the origin server"
context: "CloudFront-to-origin connections can use HTTP or HTTPS. When using HTTPS, CloudFront validates the origin certificate against trusted CAs. The origin certificate must match the origin domain name. Self-signed certificates are NOT trusted. Origin protocol policy controls whether CloudFront uses HTTP, HTTPS, or matches the viewer protocol."
---

## Phase 1 — Triage

MUST:
- Check origin protocol policy: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].CustomOriginConfig.OriginProtocolPolicy'`
- Check origin SSL protocols: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].CustomOriginConfig.OriginSslProtocols'`
- Test origin SSL directly: `openssl s_client -connect <origin-domain>:<port> -servername <origin-domain>`
- Verify origin certificate is valid and not expired
- Check if origin certificate matches the origin domain name

SHOULD:
- Verify origin supports the SSL protocols configured in CloudFront
- Check if origin uses a self-signed certificate (not supported)
- Verify the full certificate chain is served by the origin

MAY:
- Check origin certificate with: `echo | openssl s_client -connect <origin>:443 2>/dev/null | openssl x509 -noout -dates -subject`
- Test specific TLS versions: `openssl s_client -connect <origin>:443 -tls1_2`

## Phase 2 — Remediate

MUST:
- Use a certificate from a trusted CA on the origin (not self-signed)
- Ensure origin certificate covers the origin domain name
- Match origin SSL protocols to what the origin supports

SHOULD:
- Use TLSv1.2 minimum for origin connections
- Ensure origin serves the full certificate chain (including intermediates)
- Use HTTPS-only origin protocol policy for security

MAY:
- Use ACM for ALB origins (free, auto-renewing)
- Implement certificate monitoring for origin certificates

## Common Issues

- symptoms: "502 Bad Gateway after enabling HTTPS to origin"
  diagnosis: "Origin has a self-signed certificate or certificate does not match domain."
  resolution: "Install a valid certificate from a trusted CA on the origin."

- symptoms: "502 with origin using TLSv1.3 only"
  diagnosis: "CloudFront origin SSL protocols may not include TLSv1.3."
  resolution: "Ensure origin supports TLSv1.2 as well, or update CloudFront origin SSL protocols."

- symptoms: "Intermittent 502 errors on HTTPS origin"
  diagnosis: "Origin certificate is expiring or intermediate certificate is missing."
  resolution: "Renew origin certificate and ensure full chain is served."


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
3. If origin configuration was changed, restore original origin settings including timeouts and protocols
## Output Format

```yaml
root_cause: "origin_ssl — <specific_cause>"
evidence:
  - type: origin_protocol
    content: "<origin protocol policy and SSL protocols>"
  - type: origin_certificate
    content: "<origin certificate details>"
  - type: ssl_test
    content: "<openssl test result>"
severity: HIGH
mitigation:
  immediate: "Fix origin certificate or protocol configuration"
  long_term: "Implement certificate monitoring and auto-renewal"
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
