# ACM Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ACM issue.

## Guardrail 1: CloudFront Requires us-east-1 Certificates
Certificates used with Amazon CloudFront MUST be provisioned in the us-east-1 (N. Virginia) region. Certificates from any other region will not appear in the CloudFront console and cannot be associated with distributions. This is the most common ACM integration error.

## Guardrail 2: DNS Validation CNAME Must Persist for Renewal
ACM auto-renews DNS-validated certificates by re-checking the CNAME validation record. If the CNAME record is removed from DNS after initial validation, auto-renewal will fail silently. The CNAME must remain in DNS for the lifetime of the certificate.

## Guardrail 3: Email-Validated Certificates Do NOT Auto-Renew
Email-validated certificates require manual approval for every renewal cycle. ACM sends renewal emails 45 days before expiry. If no one approves, the certificate expires. DNS validation is always preferred for automatic renewal.

## Guardrail 4: Imported Certificates Are NOT Auto-Renewed
ACM does not manage the lifecycle of imported certificates. You must track expiry externally and re-import the renewed certificate before it expires. Use CloudWatch DaysToExpiry metric and set alarms at 30, 14, and 7 days.

## Guardrail 5: Wildcard Certs Do NOT Cover the Apex Domain
A wildcard certificate (*.example.com) covers single-level subdomains (www.example.com, api.example.com) but does NOT cover the apex domain (example.com) or multi-level subdomains (a.b.example.com). Add the apex domain as a Subject Alternative Name (SAN) if needed.

## Guardrail 6: Validation Must Complete Within 72 Hours
DNS and email validation must succeed within 72 hours of the certificate request. After 72 hours, the request times out and the certificate status changes to FAILED. You must request a new certificate — the failed one cannot be retried.

## Guardrail 7: Certificate In Use Cannot Be Deleted
ACM prevents deletion of certificates associated with AWS services (CloudFront, ELB, API Gateway, Elastic Beanstalk, etc.). You must disassociate the certificate from ALL services before it can be deleted. Use `describe-certificate` to check the InUseBy field.

## Guardrail 8: Public ACM Certificates Cannot Be Exported
Public certificates issued by ACM cannot be exported or downloaded. They can only be used with integrated AWS services. If you need the private key, use ACM Private CA or import your own certificate.

## Guardrail 9: Private CA Certificates Are NOT Browser-Trusted
Certificates issued by ACM Private CA are NOT trusted by public browsers or operating systems. Clients must have the private CA root certificate installed in their trust store. Do not use private CA certificates for public-facing websites.

## Guardrail 10: Key Algorithm Must Match Service Requirements
CloudFront only supports RSA 2048 and EC prime256v1 (P-256) certificates. RSA 3072, RSA 4096, and EC secp384r1 certificates will NOT work with CloudFront. Always verify the target service's key algorithm requirements before requesting a certificate.

## Guardrail 11: SAN Limit is 10 by Default
ACM certificates support up to 10 Subject Alternative Names (SANs) by default, including the primary domain. A quota increase to 100 SANs can be requested through AWS Support. Each SAN requires its own validation (DNS CNAME or email).

## Guardrail 12: Certificate Transparency Logging is Mandatory for Public Certs
All public ACM certificates are logged to Certificate Transparency (CT) logs. This cannot be disabled. CT logging means all domain names on the certificate are publicly visible. Use ACM Private CA if domain names must remain confidential.
