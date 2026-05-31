---
title: "C1 — SSL/TLS Certificate Errors"
description: "Diagnose certificate-related errors for CloudFront distributions"
status: active
severity: CRITICAL
triggers:
  - "certificate error"
  - "SSL error"
  - "TLS error"
  - "ERR_CERT"
  - "certificate expired"
  - "certificate mismatch"
owner: devops-agent
objective: "Resolve SSL/TLS certificate issues for CloudFront viewer connections"
context: "CloudFront requires ACM certificates in us-east-1 for custom domain names (CNAMEs). The certificate must cover all alternate domain names on the distribution. Common issues include certificates in wrong region, expired certificates, domain name mismatch, and DNS validation failures."
---

## Phase 1 — Triage

MUST:
- Check distribution certificate: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.ViewerCertificate'`
- Check certificate details: `aws acm describe-certificate --certificate-arn <cert-arn> --region us-east-1`
- Verify certificate covers all CNAMEs on the distribution
- Check certificate status (ISSUED, PENDING_VALIDATION, EXPIRED, FAILED)
- Check distribution alternate domain names: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Aliases'`

SHOULD:
- Verify DNS CNAME/alias points to the CloudFront distribution domain
- Check certificate renewal status for ACM-managed certificates
- Test SSL connection: `openssl s_client -connect <domain>:443 -servername <domain>`

MAY:
- Check Certificate Transparency logs for the domain
- Verify CAA records allow ACM: `dig CAA <domain>`

## Phase 2 — Remediate

MUST:
- Ensure ACM certificate is in us-east-1 (required for CloudFront)
- Add all alternate domain names to the certificate (including www variants)
- Complete DNS validation for pending certificates
- Replace expired certificates

SHOULD:
- Use ACM-managed certificates for automatic renewal
- Enable OCSP stapling (enabled by default with ACM)
- Use SNI (recommended) instead of dedicated IP for cost savings

MAY:
- Set up CloudWatch alarms for certificate expiry: `aws acm describe-certificate --certificate-arn <arn> --query 'Certificate.NotAfter'`
- Implement certificate monitoring automation

## Common Issues

- symptoms: "ERR_CERT_COMMON_NAME_INVALID in browser"
  diagnosis: "Certificate does not cover the domain name used to access CloudFront."
  resolution: "Add the domain as a SAN to the ACM certificate or request a new certificate."

- symptoms: "Certificate in PENDING_VALIDATION status"
  diagnosis: "DNS validation CNAME record not created or not propagated."
  resolution: "Create the DNS validation CNAME record. Wait for DNS propagation (up to 72 hours)."

- symptoms: "Cannot associate certificate with distribution"
  diagnosis: "Certificate is not in us-east-1 region."
  resolution: "Request a new certificate in us-east-1. CloudFront only accepts us-east-1 certificates."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
root_cause: "certificate — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<ACM certificate details>"
  - type: distribution_aliases
    content: "<alternate domain names>"
  - type: ssl_test
    content: "<openssl connection test result>"
severity: CRITICAL
mitigation:
  immediate: "Fix certificate or DNS configuration"
  long_term: "Use ACM-managed certificates with automatic renewal monitoring"
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
