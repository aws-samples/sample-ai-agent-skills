---
title: "E2 — Custom Domain Issues"
description: "Diagnose Cognito custom domain configuration and certificate problems"
status: active
severity: MEDIUM
triggers:
  - "custom domain"
  - "domain certificate"
  - "auth domain"
  - "domain not working"
  - "SSL certificate cognito"
owner: devops-agent
objective: "Identify and fix custom domain configuration for Cognito hosted UI"
context: "Custom domains require an ACM certificate in us-east-1, DNS CNAME/alias record, and proper domain ownership. The certificate must cover the exact domain. CloudFront distribution is created automatically."
---

## Phase 1 — Triage

MUST:
- Check domain status: `aws cognito-idp describe-user-pool-domain --domain <custom-domain>`
- Check ACM certificate status (must be in us-east-1): `aws acm describe-certificate --certificate-arn <cert-arn> --region us-east-1`
- Verify DNS resolution: `nslookup <custom-domain>` or `dig <custom-domain>`
- Check if the certificate covers the domain

SHOULD:
- Verify the CNAME/alias record points to the Cognito CloudFront distribution
- Check certificate validation status (DNS or email validation)
- Verify no other Cognito user pool is using the same domain

MAY:
- Check CloudFront distribution status associated with the domain
- Verify domain ownership

## Phase 2 — Remediate

MUST:
- Create ACM certificate in us-east-1 if missing
- Complete certificate validation (DNS or email)
- Create DNS CNAME/alias record pointing to the Cognito distribution

SHOULD:
- Wait for DNS propagation (up to 48 hours for new records)
- Ensure the certificate is not expired

MAY:
- Set up certificate renewal monitoring

## Common Issues

- symptoms: "Custom domain creation fails"
  diagnosis: "ACM certificate is not in us-east-1 or not validated."
  resolution: "Create and validate the certificate in us-east-1. Certificate must be ISSUED status."

- symptoms: "Custom domain not resolving"
  diagnosis: "DNS CNAME/alias record is missing or incorrect."
  resolution: "Create a CNAME record pointing to the CloudFront distribution domain from describe-user-pool-domain."

- symptoms: "SSL certificate error when accessing custom domain"
  diagnosis: "Certificate doesn't cover the domain or has expired."
  resolution: "Ensure the ACM certificate covers the exact domain name. Renew if expired."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create ACM certificate | YELLOW | New resource creation; can be deleted |
| Create DNS CNAME record | YELLOW | DNS change; recoverable |
| Complete certificate validation | GREEN | Validation action; no destructive impact |

## Escalation Conditions

- Production user pool domain changes
- DNS record changes for production domains
- Certificate changes affecting active authentication flows

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool-domain` | LOW | Domain configuration |
| `describe-certificate` | LOW | Certificate metadata |
| DNS resolution checks | LOW | Public DNS information |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER delete a custom domain without confirming no active users depend on it

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| ACM certificate creation | Delete certificate if not in use |
| DNS record change | Revert DNS record to previous value |
| Domain configuration | Delete custom domain and revert to prefix domain |

## Output Format

```yaml
root_cause: "custom_domain_failure — <specific_cause>"
evidence:
  - type: domain_status
    content: "<domain configuration>"
  - type: certificate_status
    content: "<ACM certificate details>"
severity: MEDIUM
mitigation:
  immediate: "Fix certificate or DNS configuration"
  long_term: "Set up certificate expiry monitoring and auto-renewal"
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
  - command: "describe-user-pool"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-users"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "admin-get-user"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling MFA for troubleshooting"
  - "NEVER expose user PII in investigation output"
  - "NEVER suggest removing password policies"
