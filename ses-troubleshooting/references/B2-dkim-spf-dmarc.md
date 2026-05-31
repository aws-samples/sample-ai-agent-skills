---
title: "B2 — DKIM/SPF/DMARC"
description: "Diagnose email authentication failures"
status: active
severity: HIGH
triggers:
  - "DKIM failed"
  - "SPF failed"
  - "DMARC failed"
  - "authentication error"
owner: devops-agent
objective: "Resolve DKIM, SPF, and DMARC authentication issues"
context: "Email authentication prevents spoofing and improves deliverability. DKIM requires three CNAME records. SPF alignment requires custom MAIL FROM domain. DMARC builds on both DKIM and SPF alignment."
---

## Phase 1 — Triage

MUST:
- Check DKIM status: `aws sesv2 get-email-identity --email-identity <domain> --query '{DkimStatus:DkimAttributes.Status,DkimTokens:DkimAttributes.Tokens}'`
- Check MAIL FROM configuration: `aws sesv2 get-email-identity --email-identity <domain> --query 'MailFromAttributes'`
- Verify DNS records for DKIM CNAMEs: `dig CNAME <token1>._domainkey.<domain>`
- Check SPF record for custom MAIL FROM domain: `dig TXT <mailfrom-subdomain>.<domain>`

SHOULD:
- Verify DMARC record: `dig TXT _dmarc.<domain>`
- Check all three DKIM CNAME records are present
- Verify MAIL FROM domain MX record points to SES

MAY:
- Use email header analysis to check authentication results
- Test with external DMARC validation tools

## Phase 2 — Remediate

MUST:
- Add all three DKIM CNAME records to DNS
- Configure custom MAIL FROM domain for SPF alignment
- Add SPF record for custom MAIL FROM: `v=spf1 include:amazonses.com ~all`

SHOULD:
- Add DMARC record: `v=DMARC1; p=quarantine; rua=mailto:dmarc@<domain>`
- Add MX record for custom MAIL FROM domain: `feedback-smtp.<region>.amazonses.com`
- Start with DMARC p=none for monitoring before enforcing

MAY:
- Enable DKIM signing with custom domain keys (BYODKIM)
- Set up DMARC aggregate report processing

## Common Issues

- symptoms: "DKIM status shows PENDING"
  diagnosis: "DNS CNAME records not yet propagated or missing."
  resolution: "Verify all three CNAME records. DNS propagation can take up to 72 hours."

- symptoms: "SPF passes but DMARC fails"
  diagnosis: "SPF passes for amazonses.com, not the sender domain. No SPF alignment."
  resolution: "Configure custom MAIL FROM domain for SPF alignment."

- symptoms: "DKIM verification failed after working previously"
  diagnosis: "DNS records were accidentally removed or modified."
  resolution: "Verify CNAME records still exist in DNS. Re-add if missing."

## Output Format

```yaml
root_cause: "authentication — <specific_cause>"
evidence:
  - type: dkim_status
    content: "<DKIM verification state and tokens>"
  - type: spf_dmarc
    content: "<SPF and DMARC DNS records>"
severity: HIGH
mitigation:
  immediate: "Fix DNS records for DKIM, SPF, and DMARC"
  long_term: "Monitor authentication pass rates and implement DMARC enforcement"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Email content may contain customer PII and business communications
- Recipient lists contain personal email addresses
- Bounce and complaint data reveals delivery issues with specific addresses
- Classification: HIGHLY CONFIDENTIAL - email content and recipient PII

## Prohibited Actions

- NEVER send test emails to real customer addresses
- NEVER modify production DKIM/SPF/DMARC records without email team approval
- NEVER disable bounce or complaint handling in production
- NEVER remove addresses from the suppression list without investigating the original cause

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling DKIM/SPF/DMARC"
  - "NEVER suggest removing sending limits"
  - "NEVER suggest disabling bounce/complaint handling"
