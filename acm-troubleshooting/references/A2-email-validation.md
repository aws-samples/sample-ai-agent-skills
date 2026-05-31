---
title: "A2 — Email Validation Issues"
description: "Diagnose ACM certificate email validation failures including missing emails and approval problems"
status: active
severity: HIGH
triggers:
  - "email validation"
  - "validation email not received"
  - "email approval"
  - "admin@"
  - "hostmaster@"
owner: devops-agent
objective: "Identify why email validation is not completing and ensure approval emails are received"
context: "ACM email validation sends approval emails to up to 8 addresses: 5 common admin addresses (admin@, administrator@, hostmaster@, postmaster@, webmaster@) at the domain, plus up to 3 addresses from WHOIS. Email validation does NOT support automatic renewal — every renewal requires manual email approval."
---

## Phase 1 — Triage

MUST:
- Check certificate status and validation method: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,Type:Type,DomainValidationOptions:DomainValidationOptions}'`
- Confirm validation method is EMAIL: check `ValidationMethod` in DomainValidationOptions
- Identify which email addresses ACM is sending to: check `ValidationEmails` in DomainValidationOptions
- Verify the domain's MX records are configured: `dig MX <domain>`

SHOULD:
- Check if the approval email was already sent: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions[].ValidationStatus'`
- Verify spam/junk filters are not blocking emails from `no-reply@certificates.amazon.com`
- Check if the domain has WHOIS privacy enabled (reduces available email addresses)
- Resend validation email if needed: `aws acm resend-validation-email --certificate-arn <cert-arn> --domain <domain> --validation-domain <validation-domain>`

MAY:
- Check if the validation domain override is needed for subdomains
- Review CloudTrail for ResendValidationEmail events

## Phase 2 — Remediate

MUST:
- Ensure at least one of the 5 admin addresses can receive email: admin@, administrator@, hostmaster@, postmaster@, webmaster@ at the domain
- If emails are not being received, check MX records and email routing
- If validation timed out (72 hours), delete and re-request the certificate

SHOULD:
- Consider switching to DNS validation for automatic renewal: delete the email-validated certificate and request a new one with DNS validation
- If using a subdomain, use `--validation-domain` to send emails to the parent domain
- Resend the validation email: `aws acm resend-validation-email --certificate-arn <cert-arn> --domain <domain> --validation-domain <domain>`

MAY:
- Set up email forwarding for admin addresses to ensure future validations succeed
- Document the email approval process for renewal cycles

## Common Issues

- symptoms: "Validation email not received for subdomain"
  diagnosis: "ACM sends emails to the subdomain's admin addresses, which may not exist."
  resolution: "Use --validation-domain to redirect emails to the parent domain."

- symptoms: "Email validation approved but certificate still pending"
  diagnosis: "Each SAN domain requires separate email approval."
  resolution: "Check all domains in DomainValidationOptions — each must be individually approved."

- symptoms: "Cannot switch from email to DNS validation"
  diagnosis: "Validation method cannot be changed after certificate request."
  resolution: "Delete the certificate and request a new one with DNS validation."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Ensure admin email addresses can receive mail | GREEN | No AWS resource changes, email infrastructure check |
| Resend validation email | GREEN | Non-destructive, sends a new approval email |
| Delete and re-request certificate with DNS validation | YELLOW | Deleting a pending certificate is safe, but changes validation method permanently |
| Use --validation-domain override | GREEN | Only affects where validation emails are sent |

## Escalation Conditions

- Email infrastructure changes required to receive validation emails — escalate to email/IT administrator
- Certificate approaching 72-hour timeout with no email access — escalate immediately to certificate owner
- Migration from email to DNS validation on production certificates — escalate to service owner for coordinated replacement
- Multi-SAN certificate with mixed validation status — escalate to certificate owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, validation email addresses, and service associations |
| `aws acm resend-validation-email` | LOW | Triggers email send, no sensitive data returned |
| `dig MX` | LOW | Public DNS query |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services (check InUseBy first)
- **NEVER** suggest using self-signed certificates in production as a workaround
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** approve validation emails on behalf of the domain owner without explicit authorization

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Resent validation email | No rollback needed — email approval is a manual step |
| Deleted pending certificate | Cannot undo — re-request a new certificate |
| Changed email routing/MX records | Restore original MX records from DNS backup |

## Output Format

```yaml
root_cause: "email_validation_failure — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<describe-certificate output>"
  - type: validation_emails
    content: "<list of email addresses ACM is sending to>"
severity: HIGH
mitigation:
  immediate: "Ensure admin email addresses can receive mail and approve"
  long_term: "Switch to DNS validation for automatic renewal support"
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
