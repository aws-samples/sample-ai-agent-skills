---
title: "E2 — Certificate Chain Issues"
description: "Diagnose chain of trust problems with imported or private CA certificates"
status: active
severity: HIGH
triggers:
  - "certificate chain"
  - "chain of trust"
  - "intermediate certificate"
  - "untrusted certificate"
  - "SSL chain"
  - "incomplete chain"
owner: devops-agent
objective: "Identify and resolve certificate chain of trust issues"
context: "A complete certificate chain includes the end-entity certificate, all intermediate CA certificates, and optionally the root CA certificate. Incomplete chains cause SSL/TLS errors in browsers and clients. ACM-issued public certificates include the full Amazon Trust Services chain automatically. Imported and private CA certificates require manual chain management."
---

## Phase 1 — Triage

MUST:
- Check the certificate chain: `aws acm get-certificate --certificate-arn <cert-arn>` — returns CertificateChain
- Verify the chain is complete: `openssl s_client -connect <domain>:443 -showcerts`
- Check each certificate in the chain: `openssl x509 -noout -subject -issuer -dates -in <cert-file>`
- Verify chain order: leaf certificate first, then intermediates, then root

SHOULD:
- Verify the chain validates: `openssl verify -CAfile chain.pem cert.pem`
- Check for expired certificates in the chain: any expired cert breaks the chain
- For private CA: verify the CA certificate chain is complete: `aws acm-pca get-certificate-authority-certificate --certificate-authority-arn <ca-arn>`
- Test with SSL Labs or similar tool for public-facing certificates

MAY:
- Check if the root CA is in the client's trust store
- Verify cross-signed certificates if applicable
- Check for certificate revocation (CRL or OCSP)

## Phase 2 — Remediate

MUST:
- If chain is incomplete: obtain missing intermediate certificates from the CA
- Re-import with the complete chain: `aws acm import-certificate --certificate-arn <cert-arn> --certificate fileb://cert.pem --private-key fileb://key.pem --certificate-chain fileb://full-chain.pem`
- For private CA: ensure the full CA hierarchy chain is distributed to clients

SHOULD:
- Order the chain correctly: leaf → intermediate(s) → root
- Replace expired intermediate certificates
- For ACM-issued certificates: these should have correct chains automatically — if not, contact AWS Support

MAY:
- Set up automated chain validation in CI/CD
- Monitor intermediate CA certificate expiry dates
- Document the certificate chain for each service

## Common Issues

- symptoms: "Browser shows 'certificate not trusted' but certificate is valid"
  diagnosis: "Intermediate certificate is missing from the chain."
  resolution: "Download the intermediate certificate from the CA and include it in the chain."

- symptoms: "Some clients trust the certificate, others don't"
  diagnosis: "Older clients may not have the root CA in their trust store, or chain is incomplete."
  resolution: "Include all intermediates in the chain. Verify with multiple clients."

- symptoms: "Private CA certificate not trusted by internal services"
  diagnosis: "The private CA root certificate is not installed in the service's trust store."
  resolution: "Distribute and install the private CA root certificate to all clients."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Re-import with complete chain | YELLOW | Overwrites existing certificate chain; preserves associations |
| Distribute private CA root certificate to clients | YELLOW | Changes client trust stores; affects all TLS connections |
| Replace expired intermediate certificates | YELLOW | Chain change may temporarily affect TLS validation |

## Escalation Conditions

- Production service returning "certificate not trusted" errors — escalate immediately to service team
- Intermediate CA certificate expiring — escalate to CA vendor and PKI team
- Private CA chain distribution to many clients — escalate to infrastructure team
- Chain validation failures on ACM-issued certificates — escalate to AWS Support

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm get-certificate` | HIGH | Returns full certificate and chain |
| `openssl s_client -connect -showcerts` | MEDIUM | Exposes server certificate chain |
| `openssl x509 -noout -subject -issuer` | LOW | Certificate metadata only |
| `aws acm-pca get-certificate-authority-certificate` | HIGH | Exposes CA certificate and chain |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** skip chain validation before re-importing a certificate
- **NEVER** distribute CA certificates without verifying their authenticity

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Re-imported certificate with new chain | Re-import with the previous chain if backup exists |
| Distributed private CA root to clients | Remove the CA root from client trust stores |
| Replaced intermediate certificate | Restore previous intermediate if still valid |

## Output Format

```yaml
root_cause: "chain_issue — <specific_cause>"
evidence:
  - type: chain_validation
    content: "<openssl verify output>"
  - type: chain_contents
    content: "<certificates in the chain with subjects and issuers>"
severity: HIGH
mitigation:
  immediate: "Complete the certificate chain with missing intermediates"
  long_term: "Automate chain validation and monitor intermediate CA expiry"
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
