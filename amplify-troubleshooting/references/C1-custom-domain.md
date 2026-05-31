---
title: "C1 — Custom Domain Issues"
description: "Diagnose custom domain configuration and DNS verification problems"
status: active
severity: HIGH
triggers:
  - "custom domain not working"
  - "domain not resolving"
  - "DNS verification failed"
  - "domain pending"
  - "subdomain not working"
owner: devops-agent
objective: "Identify and resolve custom domain configuration issues"
context: "Custom domain issues include: DNS verification records not created, DNS propagation delay, root domain vs subdomain configuration differences, domain stuck in pending state, or domain provider not supporting required record types."
---

## Phase 1 — Triage

MUST:
- Check domain associations: `aws amplify list-domain-associations --app-id <app-id>`
- Check specific domain status: `aws amplify get-domain-association --app-id <app-id> --domain-name <domain>`
- Verify domain status: PENDING_VERIFICATION, PENDING_DEPLOYMENT, AVAILABLE, or FAILED
- Check DNS records required by Amplify

SHOULD:
- Verify DNS records are created at the DNS provider
- Check DNS propagation: `nslookup <domain>` or `dig <domain>`
- Verify root domain has ANAME/ALIAS record (not CNAME)
- Check if domain is registered and not expired

MAY:
- Check Route 53 hosted zone if using Route 53
- Review domain association creation in CloudTrail

## Phase 2 — Remediate

MUST:
- Create required DNS verification records at DNS provider
- For root domains: create ANAME or ALIAS record (Route 53 supports this)
- For subdomains: create CNAME record pointing to Amplify
- Wait for DNS propagation (up to 48 hours)

SHOULD:
- Add domain: `aws amplify create-domain-association --app-id <app-id> --domain-name <domain> --sub-domain-settings '[{"prefix":"","branchName":"main"},{"prefix":"www","branchName":"main"}]'`
- Use Route 53 for easiest integration with Amplify

MAY:
- Set up www to root domain redirect
- Configure branch-to-subdomain mapping

## Safety Ratings

```
safety_ratings:
  - "list-domain-associations: GREEN — read-only domain check"
  - "get-domain-association: GREEN — read-only status check"
  - "create-domain-association: YELLOW — initiates DNS verification and certificate provisioning"
  - "delete-domain-association: RED — removes custom domain, causes downtime"
```

## Escalation Conditions

- Custom domain stuck in PENDING_VERIFICATION for more than 48 hours
- Domain association FAILED and needs recreation
- Root domain requires ANAME/ALIAS record not supported by DNS provider
- Domain configuration changes affect production traffic

## Data Sensitivity

- Domain association details: LOW — domain names and DNS records
- DNS verification records: LOW — CNAME values for verification
- Domain status: LOW — operational metadata

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER delete a domain association on a production app without a maintenance window
- NEVER modify DNS records without verifying the correct values from Amplify

## Phase 3 — Rollback

1. If a domain association was deleted, recreate it: `aws amplify create-domain-association --app-id <id> --domain-name <domain> --sub-domain-settings <settings>`
2. If DNS records were changed incorrectly, restore previous DNS records at the DNS provider
3. If the domain was pointed to the wrong branch, update sub-domain settings
4. Wait for DNS propagation after any DNS changes (up to 48 hours)
5. Verify the domain resolves correctly and HTTPS is working

## Output Format

```yaml
root_cause: "custom_domain_issue — <specific_cause>"
evidence:
  - type: domain_status
    content: "<domain association status>"
  - type: dns_records
    content: "<required vs actual DNS records>"
severity: HIGH
mitigation:
  immediate: "Create required DNS records and wait for propagation"
  long_term: "Use Route 53 for automated DNS management"
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
