---
title: "C2 — DNS Firewall Domain List Issues"
description: "Diagnose DNS Firewall domain list creation, import, and matching issues"
status: active
severity: MEDIUM
triggers: ["domain list not working", "domains not matching", "domain list import failed"]
owner: devops-agent
objective: "Resolve DNS Firewall domain list configuration and matching issues"
context: "Domain lists define which domains DNS Firewall rules match against. Lists can contain exact domains or wildcard patterns. Lists can be managed (AWS-provided) or custom. Import from S3 is supported for large lists."
---

## Phase 1 — Triage

MUST:
- List domain lists: `aws route53resolver list-firewall-domain-lists --query 'FirewallDomainLists[*].{Id:Id,Name:Name,DomainCount:DomainCount,Status:Status,ManagedOwnerName:ManagedOwnerName}'`
- Get domain list details: `aws route53resolver get-firewall-domain-list --firewall-domain-list-id <list-id>`
- Check domains in the list: `aws route53resolver list-firewall-domains --firewall-domain-list-id <list-id> --max-results 100`

SHOULD:
- Verify domain format (FQDN with optional wildcard prefix like *.example.com)
- Check if the domain list is referenced by any firewall rules

MAY:
- Compare domain list contents against expected domains
- Check S3 import status if domains were imported from S3

## Phase 2 — Remediate

MUST:
- Add missing domains: `aws route53resolver update-firewall-domains --firewall-domain-list-id <list-id> --operation ADD --domains <domain1> <domain2>`
- Verify domain format: exact match (example.com) or wildcard (*.example.com)
- Ensure the domain list is referenced by a firewall rule in an associated rule group

SHOULD:
- Use wildcard domains (*.example.com) to match all subdomains
- Remove incorrect domains that cause false positives

MAY:
- Import large domain lists from S3: `aws route53resolver import-firewall-domains --firewall-domain-list-id <list-id> --operation REPLACE --domain-file-url s3://<bucket>/<key>`
- Use AWS managed domain lists for common threat categories

## Common Issues

- symptoms: "Domain list has domains but firewall rule not matching queries"
  diagnosis: "Domain format mismatch — query is for subdomain but list has exact domain only."
  resolution: "Add wildcard entry *.example.com to match all subdomains of example.com."

- symptoms: "Domain list import from S3 fails"
  diagnosis: "S3 bucket permissions or domain format errors in the file."
  resolution: "Verify S3 bucket policy allows route53resolver to read. Check domain format (one per line, no trailing spaces)."

## Safety Ratings

```
safety_ratings:
  - "list-firewall-domain-lists: GREEN — read-only listing"
  - "list-firewall-domains: GREEN — read-only domain listing"
  - "update-firewall-domains ADD: YELLOW — adds domains to block/allow list"
  - "update-firewall-domains REMOVE: YELLOW — removes domains from security list"
```

## Escalation Conditions

- Domain list updates fail with service errors
- Large domain list import exceeds size limits (100,000 domains per list)
- Managed domain lists block legitimate services and cannot be modified
- Domain list changes need to propagate across multiple shared accounts

## Data Sensitivity

- `aws route53resolver list-firewall-domain-lists`: LOW — list metadata only
- `aws route53resolver list-firewall-domains`: MEDIUM — reveals blocked/allowed domains (security policy)
- S3 domain list files: MEDIUM — contain security threat intelligence
- Domain list changes: MEDIUM — modify security filtering policy

## Prohibited Actions

- NEVER delete a domain list that is referenced by active firewall rules
- NEVER use REPLACE operation on a domain list without backing up current domains
- NEVER add overly broad wildcard domains (e.g., *) that match everything
- NEVER modify AWS managed domain lists — they are read-only

## Phase 3 — Rollback

1. If domains were added incorrectly, remove them: `aws route53resolver update-firewall-domains --firewall-domain-list-id <list-id> --operation REMOVE --domains <domain>`
2. If a REPLACE operation overwrote the list, re-import from backup
3. If a domain list was deleted, recreate it and re-add domains
4. Verify firewall rules still reference the correct domain lists
5. Document domain list changes for security audit

## Output Format

```yaml
root_cause: "dns_firewall_domain_list — <specific_cause>"
evidence:
  - type: domain_list_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix domain list configuration"
  long_term: "Automate domain list management via S3 imports"
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
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"
