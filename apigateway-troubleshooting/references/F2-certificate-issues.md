---
title: "F2 — Certificate Issues"
description: "Diagnose ACM certificate problems for API Gateway custom domains"
status: active
severity: HIGH
triggers:
  - "certificate"
  - "ACM"
  - "SSL"
  - "TLS"
  - "cert"
  - "HTTPS"
  - "certificate validation"
owner: devops-agent
objective: "Resolve ACM certificate issues for API Gateway custom domains including validation, region, and renewal"
context: "API Gateway custom domains require ACM certificates. Edge-optimized endpoints require the certificate in us-east-1 (CloudFront requirement). Regional endpoints require the certificate in the same region as the API. Certificates must be in ISSUED status. DNS or email validation must be completed. Certificate renewal is automatic for DNS-validated certs but can fail if DNS records are removed."
---

## Phase 1 — Triage

MUST:
- Check certificate status: `aws acm describe-certificate --certificate-arn <cert-arn> --query '{Status:Status,Type:Type,DomainName:DomainName,SANs:SubjectAlternativeNames}'`
- Check certificate region matches endpoint type
- Check domain validation status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions'`
- Verify certificate covers the custom domain name (exact match or wildcard)
- Check certificate expiration: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.NotAfter'`

SHOULD:
- Check if DNS validation CNAME records exist: `dig CNAME _<hash>.<domain>`
- Verify no certificate renewal failures: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.RenewalSummary'`
- Check if certificate is in use by the domain: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.InUseBy'`

MAY:
- List all certificates in the region: `aws acm list-certificates --query 'CertificateSummaryList[*].{Domain:DomainName,ARN:CertificateArn,Status:Status}'`
- Check CloudTrail for certificate-related events

## Phase 2 — Remediate

MUST:
- For edge-optimized: request certificate in us-east-1: `aws acm request-certificate --domain-name <domain> --validation-method DNS --region us-east-1`
- For regional: request certificate in the API region: `aws acm request-certificate --domain-name <domain> --validation-method DNS --region <api-region>`
- Complete DNS validation by adding the CNAME record provided by ACM
- Wait for certificate status to become ISSUED before creating custom domain

SHOULD:
- Use DNS validation (not email) for automatic renewal
- Use wildcard certificates (*.example.com) for flexibility
- Ensure DNS validation records remain in place for automatic renewal

MAY:
- Import third-party certificates if not using ACM-issued certs
- Set up CloudWatch alarms for certificate expiration

## Common Issues

- symptoms: "Custom domain creation fails with 'certificate not found'"
  diagnosis: "Certificate is in wrong region for the endpoint type."
  resolution: "Edge-optimized: use us-east-1. Regional: use same region as API."

- symptoms: "Certificate stuck in PENDING_VALIDATION"
  diagnosis: "DNS or email validation not completed."
  resolution: "Add the CNAME record from DomainValidationOptions to your DNS. Wait up to 30 minutes."

- symptoms: "Certificate expired and custom domain stopped working"
  diagnosis: "Automatic renewal failed — DNS validation CNAME record was removed."
  resolution: "Re-add DNS validation CNAME record. ACM will retry renewal automatically."


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

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If integration settings were changed, restore original integration configuration and redeploy
2. If custom domain or base path mapping was changed, restore original mapping configuration
3. If CORS or method configuration was changed, restore original settings and redeploy the API
4. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "certificate — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<ACM certificate status, region, domain>"
  - type: validation
    content: "<domain validation status and records>"
  - type: expiration
    content: "<certificate expiration date>"
severity: HIGH
mitigation:
  immediate: "Fix certificate validation, region, or renewal"
  long_term: "Use DNS validation and monitor certificate expiration"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"
