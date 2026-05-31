---
title: "D2 — Private Certificate Issuance"
description: "Diagnose failures when issuing certificates from ACM Private CA"
status: active
severity: HIGH
triggers:
  - "private certificate"
  - "issue certificate PCA"
  - "private cert failed"
  - "end-entity certificate"
  - "internal certificate"
owner: devops-agent
objective: "Identify and resolve private certificate issuance failures"
context: "ACM Private CA issues private certificates for internal services, IoT devices, and custom PKI. Certificates can be issued via ACM (for AWS service integration) or directly via ACM-PCA API (for export). Common failures: CA not active, template mismatch, validity exceeds CA, IAM permissions."
---

## Phase 1 — Triage

MUST:
- Verify the CA is ACTIVE: `aws acm-pca describe-certificate-authority --certificate-authority-arn <ca-arn> --query 'CertificateAuthority.Status'`
- Check the CA certificate validity period: `aws acm-pca get-certificate-authority-certificate --certificate-authority-arn <ca-arn>`
- For ACM-issued private certs: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,FailureReason:FailureReason,CertificateAuthorityArn:CertificateAuthorityArn}'`
- For PCA-direct issuance, check CloudTrail for IssueCertificate errors

SHOULD:
- Verify IAM permissions include acm-pca:IssueCertificate and acm-pca:GetCertificate
- Check the certificate template being used matches the use case
- Verify the requested validity does not exceed the CA certificate's validity
- Check if the CA has RAM sharing configured for cross-account issuance

MAY:
- Check CA audit report for issuance history: `aws acm-pca create-certificate-authority-audit-report --certificate-authority-arn <ca-arn> --s3-bucket-name <bucket> --audit-report-response-format JSON`
- Review the CSR for correct subject and SAN fields

## Phase 2 — Remediate

MUST:
- If CA is not ACTIVE: activate it (see D1 runbook)
- If IAM permissions are missing: add acm-pca:IssueCertificate to the role/user policy
- If validity exceeds CA: reduce the certificate validity period
- Issue certificate via ACM for AWS service integration: `aws acm request-certificate --domain-name <domain> --certificate-authority-arn <ca-arn> --validation-method DNS`
- Issue certificate via PCA API for export: `aws acm-pca issue-certificate --certificate-authority-arn <ca-arn> --csr fileb://cert.csr --signing-algorithm SHA256WITHRSA --template-arn arn:aws:acm-pca:::template/EndEntityCertificate/V1 --validity Value=365,Type=DAYS`

SHOULD:
- Use appropriate templates: EndEntityCertificate for servers, EndEntityClientAuthCertificate for clients
- Retrieve the issued certificate: `aws acm-pca get-certificate --certificate-authority-arn <ca-arn> --certificate-arn <cert-arn>`
- Distribute the CA certificate chain to clients that need to trust the private certificates

MAY:
- Set up automated certificate issuance pipelines
- Configure short-lived certificates for enhanced security
- Use ACM-PCA connector for Kubernetes cert-manager integration

## Common Issues

- symptoms: "RequestCertificate fails with InvalidArn for CertificateAuthorityArn"
  diagnosis: "The CA ARN is incorrect or the CA is in a different region."
  resolution: "Verify the CA ARN and ensure it is in the same region as the ACM request."

- symptoms: "Certificate issued but clients don't trust it"
  diagnosis: "Private CA certificates are not in public trust stores."
  resolution: "Install the private CA root certificate in the client's trust store."

- symptoms: "Cross-account certificate issuance fails"
  diagnosis: "The CA is not shared via AWS RAM to the requesting account."
  resolution: "Share the CA using AWS RAM: aws ram create-resource-share --name <name> --resource-arns <ca-arn> --principals <account-id>"

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Issue certificate via ACM for AWS service integration | GREEN | Creates new certificate, non-destructive |
| Issue certificate via PCA API for export | GREEN | Creates new certificate, non-destructive |
| Fix IAM permissions for issuance | YELLOW | IAM changes have broad impact potential |
| Share CA via RAM for cross-account issuance | YELLOW | Grants cross-account certificate issuance capability |

## Escalation Conditions

- Private CA not active — escalate to PKI/security team (see D1 runbook)
- Cross-account certificate issuance required — escalate to security team for RAM sharing approval
- Certificate template selection for specialized use cases — escalate to PKI team
- CA certificate validity period insufficient for requested certificate — escalate to CA owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm-pca describe-certificate-authority` | MEDIUM | Exposes CA status and configuration |
| `aws acm-pca get-certificate-authority-certificate` | HIGH | Exposes CA certificate and chain |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names and certificate details |
| `aws acm-pca create-certificate-authority-audit-report` | HIGH | Exposes full issuance history |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production (use private CA instead)
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** issue certificates with overly broad SANs or wildcard patterns without security review
- **NEVER** export private keys to unencrypted storage

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Issued certificate via ACM | Delete the certificate: `aws acm delete-certificate` (if not in use) |
| Issued certificate via PCA API | Revoke the certificate: `aws acm-pca revoke-certificate` |
| Added IAM permissions | Remove the added IAM policy statements |
| Shared CA via RAM | Delete the RAM resource share |

## Output Format

```yaml
root_cause: "private_cert_issuance — <specific_cause>"
evidence:
  - type: ca_status
    content: "<CA status and validity>"
  - type: issuance_error
    content: "<error details from CloudTrail or API>"
severity: HIGH
mitigation:
  immediate: "Fix CA status, permissions, or certificate parameters"
  long_term: "Automate private certificate issuance and CA trust distribution"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling certificate transparency"
  - "NEVER suggest using expired certificates"
  - "NEVER suggest self-signed certs for production"
