---
title: "D1 — Private CA Issues"
description: "Diagnose ACM Private CA creation, configuration, and operational problems"
status: active
severity: HIGH
triggers:
  - "private CA"
  - "ACM-PCA"
  - "certificate authority"
  - "private certificate authority"
  - "CA creation"
  - "CA status"
owner: devops-agent
objective: "Identify and resolve ACM Private CA operational issues"
context: "ACM Private CA (ACM-PCA) allows creating private certificate authorities for internal PKI. Private CAs require activation by installing a CA certificate (self-signed for root, or signed by parent for subordinate). Common issues: CA stuck in PENDING_CERTIFICATE, CA disabled/expired, CRL configuration, and cost management."
---

## Phase 1 — Triage

MUST:
- List private CAs and their status: `aws acm-pca list-certificate-authorities --query 'CertificateAuthorities[].{Arn:Arn,Status:Status,Type:Type,CommonName:CertificateAuthorityConfiguration.Subject.CommonName}'`
- Check specific CA details: `aws acm-pca describe-certificate-authority --certificate-authority-arn <ca-arn>`
- Check CA status — must be ACTIVE to issue certificates
- If PENDING_CERTIFICATE: the CA certificate has not been installed yet

SHOULD:
- Check CRL configuration: `aws acm-pca describe-certificate-authority --certificate-authority-arn <ca-arn> --query 'CertificateAuthority.RevocationConfiguration'`
- Check the CA certificate expiry: `aws acm-pca get-certificate-authority-certificate --certificate-authority-arn <ca-arn>`
- Verify IAM permissions for ACM-PCA operations
- Check audit report: `aws acm-pca list-permissions --certificate-authority-arn <ca-arn>`

MAY:
- Check CloudTrail for PCA API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=acm-pca.amazonaws.com`
- Review CA usage and cost (Private CA has a monthly fee per CA)
- Check for CA certificate chain completeness

## Phase 2 — Remediate

MUST:
- If PENDING_CERTIFICATE (root CA): get CSR, self-sign, and install:
  ```
  aws acm-pca get-certificate-authority-csr --certificate-authority-arn <ca-arn> --output text > ca.csr
  aws acm-pca issue-certificate --certificate-authority-arn <ca-arn> --csr fileb://ca.csr --signing-algorithm SHA256WITHRSA --template-arn arn:aws:acm-pca:::template/RootCACertificate/V1 --validity Value=10,Type=YEARS
  aws acm-pca get-certificate --certificate-authority-arn <ca-arn> --certificate-arn <issued-cert-arn> --output text > ca-cert.pem
  aws acm-pca import-certificate-authority-certificate --certificate-authority-arn <ca-arn> --certificate fileb://ca-cert.pem
  ```
- If DISABLED: re-enable: `aws acm-pca update-certificate-authority --certificate-authority-arn <ca-arn> --status ACTIVE`
- If EXPIRED: the CA certificate has expired — issue a new CA certificate

SHOULD:
- Configure CRL distribution if revocation is needed
- Set up OCSP if real-time revocation checking is required
- Monitor CA certificate expiry with CloudWatch

MAY:
- Set up cross-account sharing for the private CA using RAM
- Implement CA hierarchy (root CA → subordinate CA) for security best practices
- Delete unused CAs to avoid monthly charges (must first disable, then delete)

## Common Issues

- symptoms: "Private CA stuck in PENDING_CERTIFICATE"
  diagnosis: "CA certificate has not been installed after creation."
  resolution: "Get the CSR, issue a CA certificate, and import it to activate the CA."

- symptoms: "Private CA shows DISABLED status"
  diagnosis: "CA was manually disabled or automatically disabled due to expiry."
  resolution: "Update CA status to ACTIVE if the CA certificate is still valid."

- symptoms: "High costs from Private CA"
  diagnosis: "Each private CA costs $400/month. Unused CAs still incur charges."
  resolution: "Delete unused CAs. Consider short-lived certificate mode for lower per-certificate cost."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Self-sign and install root CA certificate | RED | Activates the CA; once active, it can issue certificates |
| Re-enable disabled CA | YELLOW | Restores CA ability to issue certificates |
| Configure CRL/OCSP | GREEN | Additive configuration for revocation checking |
| Delete unused CA | RED | Permanent deletion; cannot be undone after grace period |
| Share CA via RAM | YELLOW | Grants cross-account certificate issuance capability |

## Escalation Conditions

- Private CA activation or certificate installation — escalate to PKI/security team for approval
- CA status change (enable/disable/delete) — escalate to CA owner and security team
- CA certificate approaching expiry — escalate to PKI team for renewal planning
- Cross-account CA sharing via RAM — escalate to security team for access review
- CA monthly cost concerns ($400/month per CA) — escalate to cost management team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm-pca list-certificate-authorities` | MEDIUM | Exposes CA ARNs, types, and status |
| `aws acm-pca describe-certificate-authority` | MEDIUM | Exposes CA configuration and revocation settings |
| `aws acm-pca get-certificate-authority-certificate` | HIGH | Exposes CA certificate and chain |
| `aws acm-pca list-permissions` | MEDIUM | Exposes CA access permissions |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production (except for root CA self-signing which is the standard process)
- **NEVER** suggest disabling certificate transparency logging for public certificates
- **NEVER** delete a private CA without confirming all issued certificates are accounted for
- **NEVER** share a CA via RAM without security team approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Activated CA (installed certificate) | Disable the CA: `aws acm-pca update-certificate-authority --status DISABLED` |
| Re-enabled disabled CA | Disable again: `aws acm-pca update-certificate-authority --status DISABLED` |
| Configured CRL distribution | Remove CRL configuration by updating revocation config |
| Shared CA via RAM | Delete the RAM resource share: `aws ram delete-resource-share` |
| Deleted unused CA | Cannot undo after deletion grace period |

## Output Format

```yaml
root_cause: "private_ca_issue — <specific_cause>"
evidence:
  - type: ca_status
    content: "<describe-certificate-authority output>"
  - type: ca_certificate
    content: "<CA certificate details and expiry>"
severity: HIGH
mitigation:
  immediate: "Activate or fix the private CA"
  long_term: "Implement CA monitoring and lifecycle management"
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
