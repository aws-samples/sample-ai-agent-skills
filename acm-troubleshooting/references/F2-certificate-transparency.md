---
title: "F2 — Certificate Transparency Logging"
description: "Understand and manage Certificate Transparency (CT) logging for ACM certificates"
status: active
severity: MEDIUM
triggers:
  - "certificate transparency"
  - "CT log"
  - "domain visible"
  - "certificate logged"
  - "transparency"
  - "domain privacy"
owner: devops-agent
objective: "Understand CT logging implications and manage domain visibility in public logs"
context: "All public ACM certificates are logged to Certificate Transparency (CT) logs as required by browser policies. CT logging makes domain names publicly visible. This cannot be disabled for public ACM certificates. If domain name confidentiality is required, use ACM Private CA instead."
---

## Phase 1 — Triage

MUST:
- Check if the certificate is public or private: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Type'`
- Check CT logging options: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Options'`
- Understand that AMAZON_ISSUED public certificates are ALWAYS logged to CT
- Identify all domain names on the certificate that will be publicly visible: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.SubjectAlternativeNames'`

SHOULD:
- Review whether any domain names on the certificate are sensitive or internal
- Check if CT logging opt-out was attempted (only affects pre-issuance logging, not post-issuance)
- Verify CT log entries: search crt.sh or Google CT log for the domain

MAY:
- Review organizational policy on domain name disclosure
- Check if competitors or attackers could use CT log data for reconnaissance
- Audit all public certificates for unintended domain exposure

## Phase 2 — Remediate

MUST:
- If domain names must remain confidential: use ACM Private CA instead of public ACM
- Private CA certificates are NOT logged to CT logs
- Accept that public ACM certificates will always have domains visible in CT logs

SHOULD:
- For public certificates: you can opt out of pre-issuance CT logging (but post-issuance logging still occurs): `aws acm request-certificate --domain-name <domain> --validation-method DNS --options CertificateTransparencyLoggingPreference=DISABLED`
- Review all public certificates for sensitive domain names
- Consider using generic domain names for internal services that need public certificates

MAY:
- Set up CT log monitoring to detect unauthorized certificate issuance for your domains
- Use CAA records to restrict which CAs can issue certificates for your domains
- Implement CT monitoring with tools like certspotter or Facebook CT monitoring

## Common Issues

- symptoms: "Internal domain names visible in CT logs"
  diagnosis: "Public ACM certificates are always logged to CT. Internal domains are now public."
  resolution: "Use ACM Private CA for internal domains. Cannot remove from CT logs retroactively."

- symptoms: "Want to disable CT logging for public certificate"
  diagnosis: "CT logging cannot be fully disabled for public certificates. Only pre-issuance logging can be opted out."
  resolution: "Accept CT logging for public certs, or switch to ACM Private CA for confidentiality."

- symptoms: "Unauthorized certificate detected in CT logs for our domain"
  diagnosis: "Someone else issued a certificate for your domain (possible domain hijack or misconfiguration)."
  resolution: "Investigate the issuing CA. Add CAA records to restrict issuance. Check for domain compromise."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Use ACM Private CA for confidential domains | GREEN | Creates private certificate, no public exposure |
| Opt out of pre-issuance CT logging | GREEN | Reduces CT exposure; post-issuance logging still occurs |
| Set up CT log monitoring | GREEN | Monitoring-only, no state change |
| Add CAA records | GREEN | Additive DNS change |

## Escalation Conditions

- Internal domain names discovered in CT logs — escalate to security team for exposure assessment
- Unauthorized certificate detected in CT logs for company domain — escalate immediately to security team
- Policy decision needed on CT logging vs domain confidentiality — escalate to security/compliance team
- CAA record changes affecting multiple domains — escalate to DNS administrator

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` (SubjectAlternativeNames) | MEDIUM | Exposes all domain names on the certificate |
| `aws acm describe-certificate` (Options) | LOW | CT logging preference only |
| CT log search (crt.sh) | LOW | Publicly available data |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging as a security measure (it cannot be fully disabled for public certs)
- **NEVER** claim that CT logging can be fully prevented for public ACM certificates

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Switched to Private CA | Revert to public certificate if confidentiality is no longer a concern |
| Opted out of pre-issuance CT logging | Re-enable by requesting new certificate with CT logging enabled |
| Added CAA records | Remove CAA records if no longer needed |
| Set up CT monitoring | Remove monitoring tools/subscriptions |

## Output Format

```yaml
root_cause: "certificate_transparency — <specific_cause>"
evidence:
  - type: certificate_type
    content: "<public or private>"
  - type: ct_logging
    content: "<CT logging preference>"
  - type: exposed_domains
    content: "<list of domains visible in CT logs>"
severity: MEDIUM
mitigation:
  immediate: "Use Private CA for confidential domains"
  long_term: "Implement CT log monitoring and CAA records"
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
