---
title: "C2 — SSL Certificate Issues"
description: "Diagnose SSL certificate provisioning and renewal problems"
status: active
severity: HIGH
triggers:
  - "SSL not working"
  - "certificate error"
  - "HTTPS not available"
  - "certificate pending"
  - "SSL provisioning failed"
owner: devops-agent
objective: "Identify and resolve SSL certificate issues"
context: "SSL certificate issues are tied to custom domain configuration. Amplify auto-provisions certificates via ACM. Issues include: DNS verification not complete, certificate stuck in pending, domain validation failed, or certificate renewal issues."
---

## Phase 1 — Triage

MUST:
- Check domain association and certificate status: `aws amplify get-domain-association --app-id <app-id> --domain-name <domain>`
- Verify domain status is not stuck in PENDING_VERIFICATION
- Check if DNS verification records are properly configured
- Verify the domain resolves correctly

SHOULD:
- Check ACM certificate status in us-east-1 (Amplify uses CloudFront): `aws acm list-certificates --region us-east-1 --query 'CertificateSummaryList[?DomainName==`<domain>`]'`
- Verify all subdomains have proper DNS records
- Check for CAA records that might block certificate issuance

MAY:
- Review certificate details in ACM
- Check for domain ownership conflicts

## Phase 2 — Remediate

MUST:
- Complete DNS verification by adding required records
- If stuck: delete and recreate domain association
- Ensure no CAA records block Amazon certificate issuance

SHOULD:
- Delete stuck domain: `aws amplify delete-domain-association --app-id <app-id> --domain-name <domain>`
- Recreate domain association after fixing DNS
- Add CAA record allowing Amazon: `0 issue "amazon.com"`

MAY:
- Use Route 53 for automatic DNS record management
- Monitor certificate expiration dates

## Safety Ratings

```
safety_ratings:
  - "get-domain-association: GREEN — read-only status check"
  - "acm list-certificates: GREEN — read-only certificate check"
  - "delete-domain-association: RED — removes domain and certificate, causes downtime"
  - "recreate domain association: YELLOW — initiates new certificate provisioning"
```

## Escalation Conditions

- SSL certificate stuck in pending, blocking HTTPS for production domain
- CAA records blocking Amazon certificate issuance
- Certificate renewal failure for an existing production domain
- Domain ownership conflict preventing certificate validation

## Data Sensitivity

- Domain and certificate status: LOW — operational metadata
- DNS verification records: LOW — CNAME values
- ACM certificate details: LOW — certificate metadata

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER delete a domain association to fix SSL without a maintenance window
- NEVER remove CAA records entirely — add Amazon as an allowed issuer instead

## Phase 3 — Rollback

1. If a domain association was deleted to fix SSL, recreate it immediately
2. If DNS records were changed, restore previous records and wait for propagation
3. If CAA records were modified, verify they still allow other required certificate issuers
4. Verify HTTPS is working on all subdomains after rollback
5. Monitor certificate status for the next 24 hours

## Output Format

```yaml
root_cause: "ssl_certificate_issue — <specific_cause>"
evidence:
  - type: domain_status
    content: "<domain and certificate status>"
  - type: dns_verification
    content: "<DNS record status>"
severity: HIGH
mitigation:
  immediate: "Complete DNS verification or recreate domain association"
  long_term: "Use Route 53 for automated certificate management"
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
