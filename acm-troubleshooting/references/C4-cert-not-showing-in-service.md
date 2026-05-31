---
title: "C4 — Certificate Not Showing in Service"
description: "Diagnose why an ACM certificate does not appear in the dropdown or selection list of an AWS service"
status: active
severity: MEDIUM
triggers:
  - "certificate not showing"
  - "certificate not in dropdown"
  - "certificate not available"
  - "cannot select certificate"
  - "certificate missing from"
owner: devops-agent
objective: "Identify why the certificate is not visible to the target AWS service and resolve the visibility issue"
context: "ACM certificates must meet specific criteria to appear in AWS service consoles: correct region, ISSUED status, compatible key algorithm, and correct certificate type. Each service has its own requirements."
---

## Phase 1 — Triage

MUST:
- Verify the certificate exists and is ISSUED: `aws acm list-certificates --certificate-statuses ISSUED`
- Check the certificate region matches the service region (or us-east-1 for CloudFront/edge-optimized API Gateway)
- Verify the certificate status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Status'`
- Check the certificate type (AMAZON_ISSUED vs IMPORTED vs PRIVATE): `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Type'`

SHOULD:
- Check the key algorithm compatibility with the target service: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.KeyAlgorithm'`
- Verify you are looking in the correct AWS account
- Check IAM permissions — the user/role may not have acm:ListCertificates permission
- For CloudFront: confirm you are checking us-east-1 certificates

MAY:
- Check if the certificate is a private CA certificate (some services don't support private certs)
- Verify service-linked role permissions for ACM integration
- Check AWS Config for certificate inventory across regions

## Phase 2 — Remediate

MUST:
- If wrong region: request or import the certificate in the correct region
- If not ISSUED: complete validation or resolve the certificate status issue
- If incompatible key algorithm: request a new certificate with a compatible algorithm

SHOULD:
- For CloudFront: ensure certificate is in us-east-1 with RSA 2048 or EC prime256v1
- For ALB/NLB: ensure certificate is in the same region as the load balancer
- For API Gateway edge: ensure certificate is in us-east-1
- Verify IAM permissions include acm:ListCertificates and acm:DescribeCertificate

MAY:
- Create a cross-region certificate management strategy
- Document which certificates are used by which services

## Common Issues

- symptoms: "Certificate exists but not in CloudFront dropdown"
  diagnosis: "Certificate is not in us-east-1 or key algorithm is incompatible."
  resolution: "Request certificate in us-east-1 with RSA_2048 or EC_prime256v1."

- symptoms: "Certificate not visible in ALB listener configuration"
  diagnosis: "Certificate is in a different region than the ALB."
  resolution: "Request certificate in the same region as the ALB."

- symptoms: "Certificate shows in CLI but not in console"
  diagnosis: "Console may filter by status or type. IAM permissions may differ."
  resolution: "Check console filters. Verify IAM permissions for the console user."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Request certificate in correct region | GREEN | Creates new resource, no impact on existing |
| Request certificate with compatible key algorithm | GREEN | Creates new resource, no impact on existing |
| Complete certificate validation | GREEN | Non-destructive validation step |

## Escalation Conditions

- Production service deployment blocked by certificate visibility issue — escalate to service team
- IAM permission changes required to list certificates — escalate to IAM administrator
- Cross-region certificate strategy needed — escalate to architecture team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm list-certificates` | MEDIUM | Lists all certificate ARNs and domains in the region |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, key algorithm, and service associations |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** modify IAM permissions without understanding the blast radius

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Requested certificate in new region | Delete the certificate if not needed: `aws acm delete-certificate` |
| Requested certificate with new key algorithm | Delete the certificate if not needed: `aws acm delete-certificate` |

## Output Format

```yaml
root_cause: "cert_not_showing — <specific_cause>"
evidence:
  - type: certificate_details
    content: "<certificate region, status, type, key algorithm>"
  - type: service_requirements
    content: "<target service region and compatibility requirements>"
severity: MEDIUM
mitigation:
  immediate: "Provision certificate meeting the service's requirements"
  long_term: "Document certificate requirements per service in runbooks"
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
