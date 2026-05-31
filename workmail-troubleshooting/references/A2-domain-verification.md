---
title: "A2 — Domain Verification"
description: "Diagnose WorkMail domain verification and DNS configuration issues"
status: active
severity: HIGH
triggers: ["domain not verified", "DNS records wrong", "MX records", "DKIM failed", "domain pending"]
owner: devops-agent
objective: "Resolve domain verification issues including MX, TXT, and DKIM CNAME record configuration"
context: "Custom domains require MX records for mail routing, TXT for ownership verification, and CNAME for DKIM. All three must be correct."
---

## Phase 1 — Triage

MUST:
- Check domain status: `aws workmail list-mail-domains --organization-id <org-id>`
- Get domain details: `aws workmail get-mail-domain --organization-id <org-id> --domain-name <domain>`
- Verify MX records: `dig MX <domain>` or `nslookup -type=MX <domain>`
- Verify TXT records: `dig TXT <domain>` or `nslookup -type=TXT <domain>`

SHOULD:
- Check DKIM CNAME records: `dig CNAME <selector>._domainkey.<domain>`
- Verify DNS propagation has completed (may take up to 72 hours)
- Check if the domain is registered with Route 53 or external DNS provider

MAY:
- Use Route 53 to verify record sets: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id>`
- Check for conflicting MX records from other email services

## Phase 2 — Remediate

MUST:
- Create correct MX record pointing to WorkMail: `10 inbound-smtp.<region>.amazonaws.com`
- Create TXT record for domain ownership verification
- Create DKIM CNAME records as provided by WorkMail

SHOULD:
- Remove conflicting MX records from other email services
- Set appropriate TTL values (300 seconds recommended during setup)
- Verify all records after creation

MAY:
- Set up SPF record for additional email authentication
- Configure DMARC policy for domain protection

## Safety Ratings

```
safety_ratings:
  - "workmail list-mail-domains: GREEN — read-only domain listing"
  - "workmail get-mail-domain: GREEN — read-only domain inspection"
  - "dig/nslookup: GREEN — read-only DNS query"
  - "DNS record changes: RED — affects email routing for the entire domain"
```

## Escalation Conditions

- Domain verification fails after 72 hours with correct DNS records
- MX records are correct but email is not being delivered
- DKIM verification fails despite correct CNAME records
- Domain is managed by another team and DNS changes cannot be made

## Data Sensitivity

- `aws workmail list-mail-domains`: LOW — returns domain names and status
- `aws workmail get-mail-domain`: LOW — returns domain configuration
- DNS records: LOW — publicly queryable information
- `aws route53 list-resource-record-sets`: MEDIUM — reveals all DNS records for the zone

## Prohibited Actions

- NEVER delete MX records for a domain that is actively receiving email
- NEVER change DNS records without a rollback plan
- NEVER remove the TXT verification record after domain is verified — it may be re-checked
- NEVER configure MX records for a domain you do not own

## Phase 3 — Rollback

1. If MX records were changed incorrectly, restore original MX records immediately
2. If the domain was removed from WorkMail, re-add it and reconfigure DNS
3. If DKIM records were deleted, recreate them from the WorkMail console
4. Lower DNS TTL before making changes so rollback propagates quickly
5. Monitor email delivery for 24 hours after any DNS changes

## Common Issues

- symptoms: "Domain shows 'Pending Verification' status"
  diagnosis: "TXT verification record is missing or incorrect in DNS."
  resolution: "Add the TXT record provided by WorkMail to your DNS. Wait up to 72 hours for verification."

- symptoms: "Email not delivered to custom domain"
  diagnosis: "MX record is missing or pointing to wrong endpoint."
  resolution: "Create MX record: 10 inbound-smtp.<region>.amazonaws.com. Remove conflicting MX records."

## Output Format

```yaml
root_cause: "domain_verification — <specific_cause>"
evidence:
  - type: dns_issue
    content: "<DNS records and domain status>"
severity: HIGH
mitigation:
  immediate: "Fix DNS records"
  long_term: "Monitor domain verification and DKIM status"
```

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
