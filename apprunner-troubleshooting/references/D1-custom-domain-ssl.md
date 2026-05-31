---
title: "D1 — App Runner Custom Domain and SSL Issues"
description: "Diagnose custom domain association, DNS validation, and SSL certificate problems"
status: active
severity: MEDIUM
triggers:
  - "custom domain"
  - "SSL certificate"
  - "domain validation"
  - "CNAME record"
  - "certificate pending"
  - "domain not working"
owner: devops-agent
objective: "Identify and fix custom domain association and SSL certificate validation issues"
context: "App Runner manages SSL certificates for custom domains automatically. Users must add CNAME validation records to their DNS. App Runner does NOT use user-provided ACM certificates. The default *.awsapprunner.com domain always has SSL. Domain validation can take up to 48 hours. Common issues include missing DNS records, incorrect CNAME values, and DNS propagation delays."
---

## Phase 1 — Triage

MUST:
- List custom domains and their status: `aws apprunner list-custom-domains --service-arn <service-arn>`
- Check the validation records that need to be added to DNS
- Verify DNS records are correctly configured: `nslookup -type=CNAME <custom-domain>` or `dig CNAME <custom-domain>`
- Check the certificate validation status in the custom domain response
- Verify the service is in RUNNING status: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.Status'`

SHOULD:
- Check if all required CNAME records are present (validation record AND the domain CNAME pointing to the App Runner URL)
- Verify DNS propagation has completed (can take up to 48 hours)
- Check if the domain is registered and the DNS zone is properly configured

MAY:
- Check Route 53 hosted zone if using Route 53: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id>`
- Verify no conflicting DNS records (A records that override the CNAME)

## Phase 2 — Remediate

MUST:
- Add the CNAME validation record provided by App Runner to your DNS
- Add a CNAME record pointing the custom domain to the App Runner service URL (*.awsapprunner.com)
- Wait for DNS propagation and certificate validation (up to 48 hours)

SHOULD:
- Use low TTL values for DNS records during initial setup to speed up propagation
- Verify both the validation CNAME and the domain CNAME are present
- Do NOT create an ACM certificate — App Runner manages certificates automatically

MAY:
- Disassociate and re-associate the domain if validation is stuck after 48 hours
- Use Route 53 for DNS management for faster propagation within AWS

## Common Issues

- symptoms: "Custom domain shows 'pending_certificate_dns_validation'"
  diagnosis: "CNAME validation record has not been added to DNS or has not propagated."
  resolution: "Add the exact CNAME validation record from list-custom-domains output to your DNS provider."

- symptoms: "Custom domain returns SSL error or certificate mismatch"
  diagnosis: "Certificate validation is not complete. Domain is resolving but certificate is not yet provisioned."
  resolution: "Wait for certificate validation to complete. Verify CNAME records are correct."

- symptoms: "User created an ACM certificate for the custom domain"
  diagnosis: "Misunderstanding — App Runner does not use user-provided ACM certificates."
  resolution: "Remove the ACM certificate (not needed). Use App Runner's associate-custom-domain and add the provided CNAME records."

## Output Format

```yaml
root_cause: "custom_domain — <specific_cause>"
evidence:
  - type: domain_status
    content: "<custom domain status and validation state>"
  - type: dns_records
    content: "<DNS lookup results>"
  - type: validation_records
    content: "<required CNAME records>"
severity: MEDIUM
mitigation:
  immediate: "Add missing DNS records and wait for validation"
  long_term: "Document the domain setup process and use Route 53 for faster propagation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add CNAME validation record to DNS | YELLOW | DNS configuration change |
| Add CNAME pointing to App Runner URL | YELLOW | DNS configuration change |
| Wait for DNS propagation | GREEN | Passive wait |
| Disassociate and re-associate domain | YELLOW | Domain configuration change |
| Use Route 53 for DNS | GREEN | DNS provider recommendation |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Custom domain**: Domain name and validation status — INTERNAL
- **DNS records**: CNAME records and validation records — INTERNAL
- **SSL certificate**: Certificate validation state — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER delete DNS records for a domain that is actively serving traffic

## Phase 3 — Rollback

1. Disassociate the custom domain: `aws apprunner disassociate-custom-domain --service-arn <arn> --domain-name <domain>`
2. Remove CNAME records from DNS
3. Verify the default *.awsapprunner.com domain still works
4. Re-associate the domain with correct configuration if needed
5. Communicate domain change to affected users

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
