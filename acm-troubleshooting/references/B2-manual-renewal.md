---
title: "B2 — Manual Renewal Process"
description: "Guide for manually renewing ACM certificates including email-validated and edge cases"
status: active
severity: HIGH
triggers:
  - "manual renewal"
  - "email renewal"
  - "renew certificate"
  - "certificate expiring email"
  - "renewal approval"
owner: devops-agent
objective: "Complete manual certificate renewal before expiry"
context: "Email-validated ACM certificates require manual approval for every renewal. ACM sends renewal emails 45 days before expiry to the domain's admin addresses. If not approved within the renewal window, the certificate expires. DNS-validated certificates that fail auto-renewal may also need manual intervention."
---

## Phase 1 — Triage

MUST:
- Check certificate details and expiry: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,NotAfter:NotAfter,RenewalSummary:RenewalSummary,DomainValidationOptions:DomainValidationOptions}'`
- Identify validation method: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions[].ValidationMethod'`
- For email validation: check which addresses receive renewal emails from ValidationEmails
- Check renewal eligibility: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.RenewalEligibility'`

SHOULD:
- Calculate days until expiry from NotAfter
- Check if renewal emails were sent (45 days before expiry for email validation)
- Verify email addresses in ValidationEmails are monitored
- Check if the certificate is associated with services: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.InUseBy'`

MAY:
- Check CloudTrail for any renewal-related events
- Review Personal Health Dashboard for renewal notifications

## Phase 2 — Remediate

MUST:
- For email-validated certificates: approve the renewal email sent to admin addresses
- If renewal email was not received, resend: `aws acm resend-validation-email --certificate-arn <cert-arn> --domain <domain> --validation-domain <domain>`
- If certificate is close to expiry and renewal is stuck, plan for replacement

SHOULD:
- Consider migrating from email validation to DNS validation for future auto-renewal
- To migrate: request a new certificate with DNS validation, associate it with services, then delete the old one
- Set up monitoring for future renewals

MAY:
- Automate renewal monitoring with CloudWatch alarms on DaysToExpiry
- Document the renewal approval process and responsible team

## Common Issues

- symptoms: "Renewal email not received"
  diagnosis: "Email may be in spam, or admin addresses don't exist for the domain."
  resolution: "Check spam folders. Ensure admin@, hostmaster@, etc. can receive mail. Use resend-validation-email."

- symptoms: "Renewed certificate but services still using old one"
  diagnosis: "ACM renewal updates the certificate in-place — services should pick it up automatically."
  resolution: "If not updated, check the service configuration. The certificate ARN stays the same after renewal."

- symptoms: "Want to switch from email to DNS validation"
  diagnosis: "Cannot change validation method on existing certificate."
  resolution: "Request a new certificate with DNS validation, associate with services, delete old certificate."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Approve renewal email | GREEN | Manual approval, no infrastructure change |
| Resend validation email | GREEN | Non-destructive email action |
| Migrate to DNS validation (new cert + associate + delete old) | YELLOW | Requires coordinated certificate replacement on services |
| Set up renewal monitoring | GREEN | Monitoring-only, no state change |

## Escalation Conditions

- Certificate expiry within 7 days and renewal email not received — escalate immediately to certificate owner and email administrator
- Migration from email to DNS validation on production services — escalate to service owner for coordinated replacement
- Multiple certificates requiring manual renewal simultaneously — escalate to certificate management team
- Renewal email addresses no longer monitored — escalate to IT/email administrator

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, validation emails, and service associations |
| `aws acm resend-validation-email` | LOW | Triggers email send |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services without a replacement ready
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** let a production certificate expire without escalation

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Approved renewal email | Cannot undo — renewal proceeds automatically |
| Migrated to DNS validation (new cert) | Re-associate old certificate if still valid; delete new certificate |
| Deleted old email-validated certificate | Cannot undo — re-request if needed |

## Output Format

```yaml
root_cause: "manual_renewal_required — <specific_cause>"
evidence:
  - type: certificate_details
    content: "<describe-certificate output>"
  - type: renewal_status
    content: "<RenewalSummary and eligibility>"
severity: HIGH
mitigation:
  immediate: "Approve renewal email or resend validation"
  long_term: "Migrate to DNS validation for automatic renewal"
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
