---
title: "B1 — Auto-Renewal Failures"
description: "Diagnose ACM certificate auto-renewal failures for DNS-validated certificates"
status: active
severity: CRITICAL
triggers:
  - "renewal failed"
  - "auto-renewal"
  - "certificate expiring"
  - "renewal eligibility"
  - "PENDING_VALIDATION renewal"
owner: devops-agent
objective: "Identify why auto-renewal failed and restore the renewal pipeline before certificate expiry"
context: "ACM attempts auto-renewal 60 days before expiry for DNS-validated certificates. Renewal requires the DNS CNAME validation record to still exist. If renewal fails, ACM retries periodically but the certificate will expire if not resolved. Email-validated certificates NEVER auto-renew."
---

## Phase 1 — Triage

MUST:
- Check certificate renewal status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,NotAfter:NotAfter,RenewalSummary:RenewalSummary,Type:Type}'`
- Check the RenewalSummary for failure reason and renewal status
- Verify the validation method is DNS (email does NOT auto-renew): `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions[].ValidationMethod'`
- Check if DNS CNAME validation records still exist: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Type=='CNAME']"`
- Verify the certificate is in use: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.InUseBy'`

SHOULD:
- Check the RenewalEligibility field: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.RenewalEligibility'`
- Verify DNS CNAME values match the original validation records
- Check for CAA record changes that might block renewal: `dig CAA <domain>`
- Check CloudWatch for DaysToExpiry metric: `aws cloudwatch get-metric-statistics --namespace AWS/CertificateManager --metric-name DaysToExpiry --dimensions Name=CertificateArn,Value=<cert-arn> --start-time <start> --end-time <end> --period 86400 --statistics Minimum`

MAY:
- Check AWS Health Dashboard for ACM service events
- Review CloudTrail for RenewCertificate events
- Check Personal Health Dashboard for renewal notifications

## Phase 2 — Remediate

MUST:
- If DNS CNAME is missing: recreate it with the original values from DomainValidationOptions
- If CAA records are blocking: add `0 issue "amazon.com"` CAA record
- If certificate is email-validated: this is expected — email certs do NOT auto-renew (see B2)
- If certificate is imported: this is expected — imported certs are NOT auto-renewed (see B3)

SHOULD:
- After fixing DNS, monitor renewal status: ACM will retry automatically
- Set up CloudWatch alarm on DaysToExpiry < 30 for early warning
- Verify all SAN domains have valid CNAME records, not just the primary domain

MAY:
- Set up EventBridge rule for ACM renewal state changes: `aws events put-rule --name acm-renewal-alert --event-pattern '{"source":["aws.acm"],"detail-type":["ACM Certificate Approaching Expiration"]}'`

## Common Issues

- symptoms: "RenewalSummary shows PENDING_VALIDATION"
  diagnosis: "DNS CNAME validation record was removed or changed."
  resolution: "Recreate the CNAME record. ACM will retry renewal automatically."

- symptoms: "Certificate is ELIGIBLE for renewal but not renewing"
  diagnosis: "ACM starts renewal 60 days before expiry. It may not have started yet."
  resolution: "Check NotAfter date. If more than 60 days away, renewal hasn't started."

- symptoms: "Renewal fails with CAA error"
  diagnosis: "CAA records were added or changed since original issuance."
  resolution: "Add CAA record allowing amazon.com: 0 issue \"amazon.com\""

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Recreate DNS CNAME validation record | GREEN | Non-destructive DNS addition |
| Add CAA record for amazon.com | GREEN | Additive DNS change |
| Set up CloudWatch alarm on DaysToExpiry | GREEN | Monitoring-only, no state change |
| Set up EventBridge rule for renewal alerts | GREEN | Monitoring-only, no state change |

## Escalation Conditions

- Certificate expiry within 14 days and renewal still failing — escalate immediately to certificate owner and service team
- Production service certificate with no renewal path — escalate to service owner for emergency certificate replacement
- CAA record changes required across multiple domains — escalate to DNS administrator
- Email-validated certificate approaching expiry with no DNS migration plan — escalate to certificate owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, renewal status, and service associations |
| `aws route53 list-resource-record-sets` | MEDIUM | Exposes DNS zone configuration |
| `aws cloudwatch get-metric-statistics` (DaysToExpiry) | LOW | Numeric metric only |
| `dig CAA` | LOW | Public DNS query |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production as a renewal workaround
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** remove DNS CNAME validation records for active certificates (this breaks auto-renewal)

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Recreated CNAME validation record | Delete the CNAME if it was not previously present (caution: this breaks renewal) |
| Added CAA record | Remove the CAA record if not previously present |
| Created CloudWatch alarm | Delete the alarm: `aws cloudwatch delete-alarms --alarm-names <name>` |
| Created EventBridge rule | Delete the rule: `aws events delete-rule --name <name>` |

## Output Format

```yaml
root_cause: "auto_renewal_failure — <specific_cause>"
evidence:
  - type: renewal_summary
    content: "<RenewalSummary from describe-certificate>"
  - type: dns_validation
    content: "<CNAME record status>"
severity: CRITICAL
mitigation:
  immediate: "Restore DNS CNAME validation records"
  long_term: "Set up CloudWatch DaysToExpiry alarms and EventBridge renewal notifications"
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
