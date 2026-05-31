---
title: "B3 — Imported Certificate Expiry"
description: "Manage imported certificate lifecycle and prevent expiry-related outages"
status: active
severity: CRITICAL
triggers:
  - "imported certificate expir"
  - "imported cert renewal"
  - "re-import certificate"
  - "third-party certificate"
  - "external certificate"
owner: devops-agent
objective: "Identify expiring imported certificates and re-import renewed certificates before expiry"
context: "ACM does NOT auto-renew imported certificates. The certificate owner must obtain a renewed certificate from their CA and re-import it. Re-importing to the same ARN preserves service associations. CloudWatch DaysToExpiry metric tracks imported certificate expiry."
---

## Phase 1 — Triage

MUST:
- List imported certificates and check expiry: `aws acm list-certificates --certificate-statuses ISSUED --query 'CertificateSummaryList[?Type==`IMPORTED`].{Arn:CertificateArn,Domain:DomainName,NotAfter:NotAfter}'`
- Check specific certificate expiry: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{NotAfter:NotAfter,Type:Type,ImportedAt:ImportedAt,InUseBy:InUseBy}'`
- Verify the certificate type is IMPORTED (not AMAZON_ISSUED)
- Check which services use this certificate: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.InUseBy'`

SHOULD:
- Check CloudWatch DaysToExpiry metric: `aws cloudwatch get-metric-statistics --namespace AWS/CertificateManager --metric-name DaysToExpiry --dimensions Name=CertificateArn,Value=<cert-arn> --start-time <start> --end-time <end> --period 86400 --statistics Minimum`
- Verify CloudWatch alarms exist for DaysToExpiry
- Identify all imported certificates across all regions

MAY:
- Check if the certificate can be replaced with an ACM-issued certificate (free, auto-renewed)
- Review the certificate chain for intermediate CA expiry

## Phase 2 — Remediate

MUST:
- Obtain a renewed certificate from the original CA
- Re-import to the SAME ARN to preserve service associations: `aws acm import-certificate --certificate-arn <cert-arn> --certificate fileb://cert.pem --private-key fileb://key.pem --certificate-chain fileb://chain.pem`
- Verify the re-import succeeded: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.NotAfter'`

SHOULD:
- Set up CloudWatch alarms for future expiry monitoring: alarm on DaysToExpiry < 30, < 14, < 7
- Consider replacing imported certificates with ACM-issued certificates where possible
- Test the renewed certificate before re-importing in production

MAY:
- Automate certificate renewal and re-import with Lambda and EventBridge
- Set up SNS notifications for expiry warnings
- Create a certificate inventory with expiry tracking

## Common Issues

- symptoms: "Imported certificate expired and services are down"
  diagnosis: "No monitoring was in place for imported certificate expiry."
  resolution: "Re-import the renewed certificate immediately. Set up DaysToExpiry alarms."

- symptoms: "Re-imported certificate but service still shows old cert"
  diagnosis: "Some services cache certificates. Re-import to the same ARN should update automatically."
  resolution: "Wait a few minutes for propagation. For CloudFront, create an invalidation or wait for distribution deployment."

- symptoms: "Want to switch from imported to ACM-issued certificate"
  diagnosis: "ACM-issued certificates are free and auto-renew."
  resolution: "Request a new ACM certificate, validate it, associate with services, then delete the imported one. Note: the ARN will change."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Re-import renewed certificate to same ARN | YELLOW | Overwrites existing certificate; preserves associations but irreversible |
| Set up CloudWatch DaysToExpiry alarms | GREEN | Monitoring-only, no state change |
| Replace imported cert with ACM-issued cert | YELLOW | Changes certificate ARN; requires service re-association |

## Escalation Conditions

- Imported certificate expiry within 7 days with no renewed certificate available — escalate immediately to certificate owner and CA vendor
- Production services affected by expired certificate — escalate to service team for emergency response
- Private key for renewed certificate unavailable — escalate to security team
- Certificate chain changes from CA vendor — escalate to PKI/security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, expiry dates, and service associations |
| `aws acm list-certificates` | MEDIUM | Lists all certificate ARNs and domains |
| `aws cloudwatch get-metric-statistics` (DaysToExpiry) | LOW | Numeric metric only |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production as a temporary replacement
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** re-import a certificate without verifying the private key matches
- **NEVER** store private keys in unencrypted locations during the re-import process

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Re-imported renewed certificate | Re-import the previous certificate if still valid (requires backup of old cert + key) |
| Created CloudWatch alarms | Delete the alarms: `aws cloudwatch delete-alarms` |
| Replaced with ACM-issued certificate | Re-associate the imported certificate ARN with services if still valid |

## Output Format

```yaml
root_cause: "imported_cert_expiry — <specific_cause>"
evidence:
  - type: certificate_details
    content: "<describe-certificate with expiry date>"
  - type: services_affected
    content: "<InUseBy list>"
severity: CRITICAL
mitigation:
  immediate: "Re-import renewed certificate to the same ARN"
  long_term: "Set up CloudWatch DaysToExpiry alarms and consider switching to ACM-issued certificates"
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
