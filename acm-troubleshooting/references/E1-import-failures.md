---
title: "E1 — Certificate Import Failures"
description: "Diagnose failures when importing third-party certificates into ACM"
status: active
severity: HIGH
triggers:
  - "import certificate"
  - "import failed"
  - "certificate import error"
  - "PEM format"
  - "private key mismatch"
owner: devops-agent
objective: "Identify and resolve certificate import failures"
context: "ACM allows importing third-party certificates for use with AWS services. Import requires the certificate body, private key, and certificate chain in PEM format. Common failures: wrong format, key mismatch, expired certificate, unsupported algorithm, chain incomplete."
---

## Phase 1 — Triage

MUST:
- Verify the certificate is in PEM format (begins with -----BEGIN CERTIFICATE-----)
- Verify the private key is in PEM format (begins with -----BEGIN RSA PRIVATE KEY----- or -----BEGIN PRIVATE KEY-----)
- Check if the private key matches the certificate: `openssl x509 -noout -modulus -in cert.pem | openssl md5` and `openssl rsa -noout -modulus -in key.pem | openssl md5` — MD5 hashes must match
- Verify the certificate is not expired: `openssl x509 -noout -dates -in cert.pem`
- Check the key algorithm: `openssl x509 -noout -text -in cert.pem | grep "Public Key Algorithm"`

SHOULD:
- Verify the certificate chain is complete and in correct order (leaf → intermediate → root)
- Check the chain: `openssl verify -CAfile chain.pem cert.pem`
- Ensure the private key is not encrypted (ACM does not accept encrypted keys): `openssl rsa -in key.pem -check`
- Verify the key size meets ACM requirements (RSA 1024-4096, EC prime256v1 or secp384r1)

MAY:
- Convert from other formats if needed: `openssl pkcs12 -in cert.pfx -out cert.pem -clcerts -nokeys` and `openssl pkcs12 -in cert.pfx -out key.pem -nocerts -nodes`
- Check for Windows line endings (CRLF) that may cause issues
- Verify the certificate has not been revoked

## Phase 2 — Remediate

MUST:
- Fix PEM format issues: ensure proper headers, base64 encoding, and no extra whitespace
- If key mismatch: obtain the correct private key that matches the certificate
- If encrypted key: decrypt it: `openssl rsa -in encrypted-key.pem -out decrypted-key.pem`
- Import the certificate: `aws acm import-certificate --certificate fileb://cert.pem --private-key fileb://key.pem --certificate-chain fileb://chain.pem`

SHOULD:
- For re-import (renewal), use the existing ARN: `aws acm import-certificate --certificate-arn <existing-arn> --certificate fileb://cert.pem --private-key fileb://key.pem --certificate-chain fileb://chain.pem`
- Verify the import succeeded: `aws acm describe-certificate --certificate-arn <cert-arn>`
- Set up DaysToExpiry monitoring for the imported certificate

MAY:
- Consider replacing imported certificates with ACM-issued certificates (free, auto-renewed)
- Automate certificate import with CI/CD pipelines
- Store certificates securely in AWS Secrets Manager before import

## Common Issues

- symptoms: "Import fails with 'Could not validate the certificate'"
  diagnosis: "Certificate chain is incomplete or in wrong order."
  resolution: "Ensure chain includes all intermediates in order: leaf → intermediate → root."

- symptoms: "Import fails with 'private key does not match'"
  diagnosis: "The private key was generated for a different CSR or certificate."
  resolution: "Verify key-cert match using openssl modulus comparison."

- symptoms: "Import fails with 'unsupported certificate'"
  diagnosis: "Certificate uses an unsupported algorithm or key size."
  resolution: "ACM supports RSA 1024-4096 and EC prime256v1/secp384r1. Re-issue with supported algorithm."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix PEM format and re-import | GREEN | Corrects certificate data, non-destructive |
| Decrypt encrypted private key | YELLOW | Handles sensitive key material |
| Import certificate (new) | GREEN | Creates new resource |
| Re-import to existing ARN | YELLOW | Overwrites existing certificate; preserves associations |

## Escalation Conditions

- Private key mismatch and original key unavailable — escalate to certificate owner and CA vendor
- Production service waiting on certificate import — escalate to service team
- Certificate chain incomplete and intermediate CA unknown — escalate to CA vendor
- Imported certificate for compliance-critical service — escalate to security team for validation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `openssl x509 -noout -modulus` | HIGH | Processes certificate cryptographic material |
| `openssl rsa -noout -modulus` | HIGH | Processes private key material |
| `openssl verify -CAfile` | MEDIUM | Validates certificate chain |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names and certificate details |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** store private keys in unencrypted locations or log private key content
- **NEVER** transmit private keys over unencrypted channels

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Imported new certificate | Delete the certificate: `aws acm delete-certificate` (if not in use) |
| Re-imported to existing ARN | Re-import the previous certificate if backup exists |
| Decrypted private key | Securely delete the decrypted key file after import |

## Output Format

```yaml
root_cause: "import_failure — <specific_cause>"
evidence:
  - type: certificate_format
    content: "<format validation results>"
  - type: key_match
    content: "<modulus comparison result>"
severity: HIGH
mitigation:
  immediate: "Fix the certificate format, key, or chain and re-import"
  long_term: "Consider using ACM-issued certificates for auto-renewal"
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
