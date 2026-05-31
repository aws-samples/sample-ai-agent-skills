---
title: "C2 — Certificate Issues"
description: "Diagnose and resolve ELB certificate provisioning, binding, and renewal issues"
status: active
severity: HIGH
triggers:
  - "certificate not found"
  - "certificate expired"
  - "certificate binding"
  - "ACM certificate"
  - "SNI certificate"
owner: devops-agent
objective: "Resolve certificate provisioning and binding issues on ELB listeners"
context: "ELB uses certificates from ACM or IAM for HTTPS/TLS listeners. Certificates must be in the same region as the load balancer. ALB supports SNI (Server Name Indication) for multiple certificates on a single listener. Common issues include certificate not available in the correct region, expired certificates, failed ACM validation, and SNI misconfiguration."
---

## Phase 1 — Triage

MUST:
- List certificates on the listener: `aws elbv2 describe-listener-certificates --listener-arn <listener-arn>`
- Check certificate status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,NotAfter:NotAfter,RenewalSummary:RenewalSummary}'`
- Verify the certificate is in the same region as the ALB
- Check if the certificate covers the required domains: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{DomainName:DomainName,SANs:SubjectAlternativeNames}'`
- Verify the listener has a default certificate: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`

SHOULD:
- Check ACM validation status for pending certificates: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions'`
- Verify DNS CNAME records for ACM DNS validation
- Check if the certificate is used by other services (CloudFront, API Gateway)

MAY:
- List all certificates in the region: `aws acm list-certificates --query 'CertificateSummaryList[*].{ARN:CertificateArn,Domain:DomainName,Status:Status}'`
- Check IAM server certificates if not using ACM: `aws iam list-server-certificates`
- Review certificate transparency logs for the domain

## Phase 2 — Remediate

MUST:
- If certificate expired: request a new certificate or renew: `aws acm request-certificate --domain-name <domain> --validation-method DNS --subject-alternative-names <san1> <san2>`
- If certificate not in correct region: request a new certificate in the ALB's region
- If ACM validation pending: add the required DNS CNAME records for validation
- If certificate not bound: attach to listener: `aws elbv2 modify-listener --listener-arn <listener-arn> --certificates CertificateArn=<cert-arn>`

SHOULD:
- Enable ACM auto-renewal by ensuring DNS validation records remain in place
- Add SNI certificates for additional domains: `aws elbv2 add-listener-certificates --listener-arn <listener-arn> --certificates CertificateArn=<cert-arn>`
- Verify the certificate works after binding: `openssl s_client -connect <lb-dns>:443 -servername <domain>`

MAY:
- Set up CloudWatch alarms for certificate expiry (via ACM or custom metrics)
- Implement automated certificate management with IaC
- Document the certificate inventory and renewal schedule

## Common Issues

- symptoms: "Listener creation fails with 'certificate not found'"
  diagnosis: "Certificate ARN is incorrect or the certificate is in a different region."
  resolution: "Verify the certificate ARN and ensure it is in the same region as the ALB."

- symptoms: "ACM certificate stuck in 'Pending validation'"
  diagnosis: "DNS CNAME validation records not created or propagated."
  resolution: "Add the CNAME records from ACM to the DNS zone. Wait for propagation (up to 72 hours)."

- symptoms: "Wrong certificate served for a specific domain"
  diagnosis: "SNI certificate not configured or default certificate is being served."
  resolution: "Add the domain-specific certificate as an SNI certificate on the listener."

- symptoms: "Certificate auto-renewal failed"
  diagnosis: "DNS validation CNAME records were removed or the domain is no longer resolvable."
  resolution: "Re-add the DNS validation CNAME records. Verify domain DNS is functioning."

## Output Format

```yaml
root_cause: "Certificate issue — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<ACM certificate details>"
  - type: listener_config
    content: "<listener certificate configuration>"
severity: HIGH
mitigation:
  immediate: "Fix certificate validation, binding, or renewal"
  long_term: "Implement automated certificate management and expiry monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/list): GREEN — read-only"
  - "Phase 2 certificate binding: YELLOW — affects HTTPS traffic"
  - "Phase 2 certificate request: GREEN — creates new certificate without affecting existing"

## Escalation Conditions

escalation_conditions:
  - "Production certificate has expired with no replacement available"
  - "Certificate validation cannot be completed due to DNS issues"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Certificate change affects multiple services sharing the same certificate"
  - "Issue involves wildcard certificate covering many subdomains"

## Data Sensitivity

data_sensitivity:
  - command: "describe-certificate"
    sensitivity: MEDIUM
    contains: "Certificate details, domain names, and validation records"
  - command: "describe-listener-certificates"
    sensitivity: LOW
    contains: "Certificate ARNs associated with listeners"
  - command: "DNS validation records"
    sensitivity: LOW
    contains: "CNAME records for certificate validation"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete a certificate that is in use by a production listener"
  - "NEVER use expired certificates in production"
  - "NEVER remove DNS validation records for auto-renewing certificates"
  - "NEVER share certificate private keys (for imported certificates)"

## Phase 3 — Rollback

- Before changing listener certificates, document the current certificate ARN
- If a new certificate causes TLS errors, re-attach the previous certificate
- If SNI certificates were added incorrectly, remove them: `aws elbv2 remove-listener-certificates --listener-arn <listener-arn> --certificates CertificateArn=<cert-arn>`
- If DNS validation records were modified, restore them for auto-renewal
- Verify HTTPS traffic works correctly after any certificate changes
