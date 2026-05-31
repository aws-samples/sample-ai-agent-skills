---
title: "F1 — Key Algorithm Requirements"
description: "Diagnose key algorithm incompatibilities between ACM certificates and AWS services"
status: active
severity: HIGH
triggers:
  - "key algorithm"
  - "RSA 2048"
  - "RSA 4096"
  - "EC prime256v1"
  - "ECDSA"
  - "elliptic curve"
  - "unsupported algorithm"
owner: devops-agent
objective: "Ensure the certificate key algorithm is compatible with the target AWS service"
context: "ACM supports RSA 2048, RSA 3072, RSA 4096, EC prime256v1 (P-256), and EC secp384r1 (P-384). However, not all AWS services support all algorithms. CloudFront only supports RSA 2048 and EC prime256v1. Choosing the wrong algorithm prevents the certificate from being used with the target service."
---

## Phase 1 — Triage

MUST:
- Check the certificate's key algorithm: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.KeyAlgorithm'`
- Identify the target AWS service and its algorithm requirements
- Verify the certificate status is ISSUED: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Status'`

SHOULD:
- Check which services the certificate is currently associated with: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.InUseBy'`
- Review the key algorithm compatibility matrix (see below)
- For imported certificates: `openssl x509 -noout -text -in cert.pem | grep "Public Key Algorithm"`

MAY:
- Check if the service has recently added support for additional algorithms
- Review security requirements for the key algorithm choice

## Key Algorithm Compatibility

| Service | RSA 2048 | RSA 3072 | RSA 4096 | EC P-256 | EC P-384 |
|---------|----------|----------|----------|----------|----------|
| CloudFront | ✅ | ❌ | ❌ | ✅ | ❌ |
| ALB | ✅ | ✅ | ✅ | ✅ | ✅ |
| NLB | ✅ | ✅ | ✅ | ✅ | ✅ |
| API Gateway (Edge) | ✅ | ❌ | ❌ | ✅ | ❌ |
| API Gateway (Regional) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Elastic Beanstalk | ✅ | ✅ | ✅ | ✅ | ✅ |

## Phase 2 — Remediate

MUST:
- If algorithm is incompatible: request a new certificate with a compatible algorithm: `aws acm request-certificate --domain-name <domain> --validation-method DNS --key-algorithm RSA_2048`
- Available key algorithm values: RSA_2048, RSA_3072, RSA_4096, EC_prime256v1, EC_secp384r1
- Validate the new certificate and associate it with the service

SHOULD:
- Use RSA_2048 for maximum compatibility across all AWS services
- Use EC_prime256v1 for better performance with CloudFront compatibility
- Update IaC templates to specify the correct key algorithm

MAY:
- Consider EC algorithms for better TLS handshake performance
- Document key algorithm requirements per service in team runbooks
- Plan migration path if changing algorithms on existing services

## Common Issues

- symptoms: "Certificate not appearing in CloudFront dropdown"
  diagnosis: "Certificate uses RSA 3072, RSA 4096, or EC P-384 — not supported by CloudFront."
  resolution: "Request a new certificate with RSA_2048 or EC_prime256v1 in us-east-1."

- symptoms: "API Gateway edge-optimized endpoint rejects certificate"
  diagnosis: "Edge-optimized endpoints use CloudFront — same algorithm restrictions apply."
  resolution: "Use RSA_2048 or EC_prime256v1 for edge-optimized API Gateway."

- symptoms: "Want stronger encryption but need CloudFront compatibility"
  diagnosis: "EC prime256v1 provides equivalent security to RSA 3072 with CloudFront support."
  resolution: "Use EC_prime256v1 for strong security with CloudFront compatibility."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Request new certificate with compatible algorithm | GREEN | Creates new resource, no impact on existing |
| Update IaC templates with key algorithm | GREEN | Template change, no immediate infrastructure impact |

## Escalation Conditions

- Production service blocked by key algorithm incompatibility — escalate to service team
- Security policy requires specific key algorithm that conflicts with service requirements — escalate to security team
- Algorithm migration across multiple services — escalate to architecture team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, key algorithm, and service associations |
| `openssl x509 -noout -text` | MEDIUM | Exposes certificate details including public key |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** downgrade key algorithm strength without security team approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Requested certificate with new algorithm | Delete the new certificate if not needed: `aws acm delete-certificate` |
| Associated new certificate with service | Re-associate the previous certificate if still valid |

## Output Format

```yaml
root_cause: "key_algorithm — <specific_cause>"
evidence:
  - type: certificate_algorithm
    content: "<KeyAlgorithm from describe-certificate>"
  - type: service_requirements
    content: "<target service algorithm requirements>"
severity: HIGH
mitigation:
  immediate: "Request new certificate with compatible key algorithm"
  long_term: "Standardize on RSA_2048 or EC_prime256v1 for maximum compatibility"
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
