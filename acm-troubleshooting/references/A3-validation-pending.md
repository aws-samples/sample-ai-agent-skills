---
title: "A3 — Validation Pending / Timeout"
description: "Diagnose certificates stuck in PENDING_VALIDATION status or approaching the 72-hour timeout"
status: active
severity: HIGH
triggers:
  - "pending validation"
  - "PENDING_VALIDATION"
  - "certificate not issued"
  - "validation timeout"
  - "certificate stuck"
  - "72 hours"
owner: devops-agent
objective: "Determine why validation is not completing and resolve before the 72-hour timeout"
context: "ACM certificate requests must be validated within 72 hours. After timeout, the status changes to FAILED and cannot be retried. Common causes: DNS CNAME not created, email not approved, CAA records blocking, wrong hosted zone, DNS propagation delay."
---

## Phase 1 — Triage

MUST:
- Check certificate status and age: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,CreatedAt:CreatedAt,DomainValidationOptions:DomainValidationOptions}'`
- Determine validation method (DNS or EMAIL) from DomainValidationOptions
- For DNS: verify CNAME records exist and are correct (see A1)
- For EMAIL: verify approval emails were sent and received (see A2)
- Check each domain's individual validation status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions[].{Domain:DomainName,Status:ValidationStatus,Method:ValidationMethod}'`

SHOULD:
- Calculate time remaining before 72-hour timeout from CreatedAt
- Check for CAA records blocking issuance: `dig CAA <domain>`
- Verify the domain is resolvable: `dig A <domain>` and `dig NS <domain>`
- Check if any individual domain in a multi-SAN certificate is blocking issuance

MAY:
- Check CloudTrail for any ACM API errors
- Verify no AWS service limits are being hit: `aws service-quotas get-service-quota --service-code acm --quota-code L-F141DD1D`

## Phase 2 — Remediate

MUST:
- If DNS validation: create or fix CNAME records immediately (see A1 runbook)
- If email validation: resend and approve emails immediately (see A2 runbook)
- If certificate has FAILED status: delete it and request a new one
- If CAA records are blocking: add `0 issue "amazon.com"` CAA record

SHOULD:
- For multi-SAN certificates, validate all domains — one unvalidated domain blocks the entire certificate
- If close to 72-hour timeout and DNS is slow, consider requesting a new certificate proactively
- Monitor with: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Status'`

MAY:
- Set up CloudWatch Events rule for ACM certificate state changes
- Automate validation record creation in CI/CD pipelines

## Common Issues

- symptoms: "Certificate pending for multiple domains, some validated, some not"
  diagnosis: "Each SAN requires individual validation. One unvalidated domain blocks issuance."
  resolution: "Check DomainValidationOptions for each domain. Fix the unvalidated ones."

- symptoms: "Certificate status changed to FAILED"
  diagnosis: "72-hour validation window expired."
  resolution: "Delete the failed certificate and request a new one. CNAME values will be different."

- symptoms: "DNS CNAME created but validation still pending after hours"
  diagnosis: "DNS propagation delay or CNAME values are incorrect."
  resolution: "Verify CNAME values exactly match describe-certificate output. Check with external DNS lookup."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create/fix CNAME validation records | GREEN | Non-destructive DNS addition |
| Resend and approve validation emails | GREEN | Non-destructive email action |
| Delete FAILED certificate and re-request | YELLOW | FAILED certs cannot be recovered; new request has different CNAME values |
| Add CAA record for amazon.com | GREEN | Additive DNS change |

## Escalation Conditions

- Certificate within 12 hours of 72-hour timeout — escalate immediately to certificate owner
- Multi-SAN certificate with one domain blocking issuance — escalate to DNS administrator for all domains
- Production service waiting on certificate issuance — escalate to service team for contingency planning
- Repeated validation failures across multiple certificate requests — escalate to AWS Support

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, validation status, and timestamps |
| `dig CAA / dig NS / dig A` | LOW | Public DNS queries |
| `aws service-quotas get-service-quota` | LOW | Account quota information |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** delete a PENDING_VALIDATION certificate without confirming it is not expected by another team

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created CNAME validation records | Delete the CNAME records from the hosted zone |
| Added CAA record | Remove the CAA record if not previously present |
| Deleted FAILED certificate | Cannot undo — re-request if needed |
| Requested new certificate | Delete the new certificate if no longer needed: `aws acm delete-certificate` |

## Output Format

```yaml
root_cause: "validation_pending — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<describe-certificate output with timestamps>"
  - type: validation_details
    content: "<per-domain validation status>"
severity: HIGH
mitigation:
  immediate: "Complete validation before 72-hour timeout"
  long_term: "Automate DNS validation record creation in infrastructure-as-code"
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
