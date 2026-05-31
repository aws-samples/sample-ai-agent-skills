---
name: acm-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Certificate Manager
  (ACM) problems by analyzing certificate status, validation state,
  renewal configuration, and following structured runbooks. Activate when:
  DNS validation failures, email validation issues, certificate pending
  validation, auto-renewal failures, imported certificate expiry, CloudFront
  certificate errors (us-east-1 requirement), ALB/NLB certificate issues,
  API Gateway custom domain certificate problems, private CA issues,
  certificate import failures, chain of trust errors, key algorithm
  problems, certificate transparency logging concerns, in-use certificate
  deletion errors, orphaned certificates, or the user says something is
  wrong with ACM without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ACM, ACM-PCA, Route 53, CloudFront,
  ELB, API Gateway, CloudTrail, and CloudWatch permissions.
---

# ACM Diagnostics

## When to use

Any ACM investigation where the console alone is insufficient — certificate validation failures, renewal problems, integration issues with AWS services, private CA troubleshooting, import errors, or security and compliance concerns.

## Investigation workflow

### Step 1 — Collect and triage

```
aws acm list-certificates --certificate-statuses ISSUED PENDING_VALIDATION FAILED EXPIRED REVOKED INACTIVE
aws acm describe-certificate --certificate-arn <cert-arn>
aws acm get-certificate --certificate-arn <cert-arn>
aws acm list-tags-for-certificate --certificate-arn <cert-arn>
```

### Step 2 — Domain deep dive

```
aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions'
aws route53 list-hosted-zones
aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Type=='CNAME']"
aws acm-pca list-certificate-authorities
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=acm.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/CertificateManager --metric-name DaysToExpiry --dimensions Name=CertificateArn,Value=<cert-arn> --start-time <start> --end-time <end> --period 86400 --statistics Minimum
```

Read `references/acm-guardrails.md` before concluding on any ACM issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `acm list-certificates` | List all certificates and their statuses |
| `acm describe-certificate` | Full certificate details, validation, renewal |
| `acm get-certificate` | Retrieve certificate body and chain |
| `acm list-tags-for-certificate` | Certificate tags and metadata |
| `acm-pca list-certificate-authorities` | List private CAs |
| `acm-pca describe-certificate-authority` | Private CA details and status |
| `acm-pca get-certificate-authority-csr` | Private CA CSR for signing |
| `route53 list-resource-record-sets` | Verify DNS validation CNAME records |
| `elbv2 describe-listeners` | Check ALB/NLB certificate bindings |
| `cloudfront get-distribution` | Check CloudFront certificate config |
| `apigateway get-domain-names` | Check API Gateway custom domain certs |

## Gotchas: ACM

- DNS validation vs email validation: DNS validation is strongly preferred — it supports automatic renewal. Email validation requires manual action for every renewal and does not auto-renew.
- CloudFront requires us-east-1 certificates: Certificates used with CloudFront distributions MUST be provisioned in us-east-1, regardless of where other resources are deployed. This is the single most common ACM + CloudFront error.
- Renewal failures: ACM auto-renews DNS-validated certificates 60 days before expiry. If the DNS CNAME validation record is removed, auto-renewal fails silently. Email-validated certificates require manual approval for renewal.
- Private CA vs public: ACM public certificates are free and trusted by browsers. ACM Private CA certificates cost money, are NOT trusted by browsers, and require distributing the CA certificate to clients.
- Imported certificate expiry is NOT auto-renewed: ACM does not manage renewal for imported certificates. You must track expiry and re-import before the certificate expires. CloudWatch DaysToExpiry metric helps monitor this.
- Certificate transparency logging: All public ACM certificates are logged to Certificate Transparency (CT) logs by default. This cannot be disabled for public certificates. Private CA certificates are NOT logged to CT.
- Wildcard certificates: A wildcard cert (*.example.com) covers one subdomain level only. It does NOT cover the apex domain (example.com) or multi-level subdomains (a.b.example.com). Add the apex as a SAN if needed.
- SAN limits: ACM certificates support up to 10 Subject Alternative Names (SANs) by default. You can request a quota increase to up to 100 SANs.
- Key algorithm requirements: ACM supports RSA 2048, RSA 3072, RSA 4096, EC prime256v1, and EC secp384r1. CloudFront only supports RSA 2048 and EC prime256v1 certificates. Choosing the wrong algorithm breaks CloudFront integration.
- Certificate in use cannot be deleted: ACM prevents deletion of certificates that are associated with AWS services (CloudFront, ALB, API Gateway, etc.). You must disassociate the certificate from all services before deletion.
- Validation timeout: DNS and email validation must complete within 72 hours. After that, the certificate request expires and must be re-created.
- ACM certificates cannot be exported: Public ACM certificates cannot be exported or used outside AWS services. Only imported certificates and ACM Private CA certificates can be exported.

## Anti-hallucination rules

1. Always cite specific certificate ARNs, validation records, or API responses as evidence.
2. CloudFront certificates MUST be in us-east-1. Never suggest using a certificate from another region with CloudFront.
3. Imported certificates are NOT auto-renewed. Never claim ACM will renew an imported certificate.
4. Wildcard certs do NOT cover the apex domain. Never claim *.example.com covers example.com.
5. Email-validated certificates do NOT auto-renew. Never claim email validation supports automatic renewal.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Validation | A1-A3 | DNS validation failures, email validation issues, validation pending/timeout |
| B — Renewal | B1-B3 | Auto-renewal failures, manual renewal process, imported cert expiry |
| C — Integration | C1-C4 | CloudFront (us-east-1), ALB/NLB binding, API Gateway custom domains, cert not showing in service |
| D — Private CA | D1-D2 | Private CA issues, private certificate issuance |
| E — Import | E1-E2 | Import failures, chain of trust issues |
| F — Security | F1-F2 | Key algorithm requirements, certificate transparency |
| G — Deletion | G1-G2 | In-use certificate deletion, orphaned certificates |
| Z — Catch-All | Z1 | General ACM troubleshooting |
