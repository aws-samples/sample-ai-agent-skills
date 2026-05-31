---
title: "C1 — CloudFront Certificate (us-east-1 Requirement)"
description: "Diagnose CloudFront certificate issues caused by wrong region or incompatible certificate configuration"
status: active
severity: CRITICAL
triggers:
  - "CloudFront certificate"
  - "CloudFront SSL"
  - "CloudFront HTTPS"
  - "certificate not showing CloudFront"
  - "us-east-1"
  - "ViewerCertificate"
owner: devops-agent
objective: "Ensure the ACM certificate is in us-east-1 and compatible with CloudFront requirements"
context: "CloudFront ONLY uses ACM certificates from us-east-1 (N. Virginia). Certificates in any other region are invisible to CloudFront. Additionally, CloudFront only supports RSA 2048 and EC prime256v1 key algorithms. This is the most common ACM + CloudFront integration error."
---

## Phase 1 — Triage

MUST:
- Check the certificate region — it MUST be us-east-1: `aws acm describe-certificate --certificate-arn <cert-arn> --region us-east-1`
- If the certificate ARN contains a different region, that is the problem
- Check the CloudFront distribution's certificate config: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.DistributionConfig.ViewerCertificate'`
- Verify the certificate key algorithm: `aws acm describe-certificate --certificate-arn <cert-arn> --region us-east-1 --query 'Certificate.KeyAlgorithm'`
- Confirm the certificate status is ISSUED: `aws acm describe-certificate --certificate-arn <cert-arn> --region us-east-1 --query 'Certificate.Status'`

SHOULD:
- List all certificates in us-east-1: `aws acm list-certificates --region us-east-1 --certificate-statuses ISSUED`
- Verify the certificate's domain names match the CloudFront distribution's CNAMEs: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.DistributionConfig.Aliases'`
- Check if the certificate covers all alternate domain names (CNAMEs) on the distribution

MAY:
- Check for other distributions using the same certificate
- Verify the distribution is deployed (Status: Deployed)

## Phase 2 — Remediate

MUST:
- If certificate is in wrong region: request a new certificate in us-east-1: `aws acm request-certificate --domain-name <domain> --validation-method DNS --region us-east-1`
- If key algorithm is incompatible (RSA 3072/4096 or EC secp384r1): request a new certificate with RSA_2048 or EC_prime256v1: `aws acm request-certificate --domain-name <domain> --validation-method DNS --key-algorithm RSA_2048 --region us-east-1`
- Associate the certificate with CloudFront: `aws cloudfront update-distribution --id <dist-id> --distribution-config <config-with-new-cert>`

SHOULD:
- Ensure the certificate includes all CloudFront alternate domain names as SANs
- Wait for CloudFront distribution to deploy after certificate change (can take 5-15 minutes)
- Validate the certificate before associating with CloudFront

MAY:
- Use CloudFormation or Terraform to manage the us-east-1 certificate alongside the distribution
- Set up cross-region certificate monitoring

## Common Issues

- symptoms: "Certificate not appearing in CloudFront console dropdown"
  diagnosis: "Certificate is not in us-east-1 or is not in ISSUED status."
  resolution: "Request or import the certificate in us-east-1. Ensure it is fully validated."

- symptoms: "CloudFront returns 502 after certificate change"
  diagnosis: "Certificate domain names don't match the distribution's alternate domain names."
  resolution: "Ensure the certificate SANs cover all CNAMEs configured on the distribution."

- symptoms: "Cannot use RSA 4096 certificate with CloudFront"
  diagnosis: "CloudFront only supports RSA 2048 and EC prime256v1."
  resolution: "Request a new certificate with --key-algorithm RSA_2048 in us-east-1."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Request new certificate in us-east-1 | GREEN | Creates new resource, no impact on existing |
| Associate certificate with CloudFront distribution | YELLOW | Changes distribution config; triggers redeployment (5-15 min) |
| Request certificate with different key algorithm | GREEN | Creates new resource, no impact on existing |

## Escalation Conditions

- Production CloudFront distribution serving errors due to certificate mismatch — escalate immediately to service team
- Certificate replacement requires CloudFront distribution redeployment during peak traffic — escalate to operations team
- Multiple distributions affected by wrong-region certificate — escalate to infrastructure team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names and certificate details |
| `aws cloudfront get-distribution` | MEDIUM | Exposes distribution configuration and origins |
| `aws acm list-certificates` | MEDIUM | Lists all certificate ARNs and domains |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** modify a CloudFront distribution without confirming the replacement certificate is validated and ISSUED

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Associated new certificate with CloudFront | Re-associate the previous certificate or revert to CloudFront default certificate |
| Requested new certificate in us-east-1 | Delete the new certificate if not needed: `aws acm delete-certificate --region us-east-1` |
| Updated distribution config | Revert distribution config to previous version |

## Output Format

```yaml
root_cause: "cloudfront_certificate — <specific_cause>"
evidence:
  - type: certificate_region
    content: "<certificate ARN showing region>"
  - type: key_algorithm
    content: "<KeyAlgorithm from describe-certificate>"
  - type: distribution_config
    content: "<ViewerCertificate configuration>"
severity: CRITICAL
mitigation:
  immediate: "Provision certificate in us-east-1 with compatible key algorithm"
  long_term: "Use IaC to enforce us-east-1 certificate provisioning for CloudFront"
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
