---
title: "A1 — DNS Validation Failures"
description: "Diagnose ACM certificate DNS validation failures by verifying CNAME records and DNS propagation"
status: active
severity: HIGH
triggers:
  - "DNS validation failed"
  - "CNAME not found"
  - "validation pending"
  - "certificate not issued"
  - "DNS validation CNAME"
owner: devops-agent
objective: "Identify why DNS validation is failing and ensure the correct CNAME record is in place"
context: "ACM DNS validation requires a specific CNAME record in the domain's DNS zone. The CNAME name and value are unique per domain per certificate request. Common failures include wrong CNAME values, wrong hosted zone, CNAME not propagated, or CAA records blocking issuance."
---

## Phase 1 — Triage

MUST:
- Get the certificate details and validation records: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.DomainValidationOptions'`
- Note the required CNAME Name and Value for each domain
- Check if the CNAME exists in Route 53: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Type=='CNAME']"`
- Verify the certificate status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.Status'`
- Check for CAA records that might block issuance: `dig CAA <domain>` or `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Type=='CAA']"`

SHOULD:
- Verify DNS propagation externally: `dig CNAME _<hash>.<domain>` or `nslookup -type=CNAME _<hash>.<domain>`
- Check if the hosted zone is the authoritative zone for the domain: `dig NS <domain>`
- Verify the domain's nameservers match the Route 53 hosted zone NS records
- Check if the certificate request is still within the 72-hour validation window

MAY:
- Check CloudTrail for certificate request events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RequestCertificate`
- Verify there are no conflicting CNAME records with the same name

## Phase 2 — Remediate

MUST:
- If CNAME is missing, create it in the correct hosted zone: `aws route53 change-resource-record-sets --hosted-zone-id <zone-id> --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"_<hash>.<domain>","Type":"CNAME","TTL":300,"ResourceRecords":[{"Value":"_<hash>.acm-validations.aws."}]}}]}'`
- If CNAME values are wrong, update with the correct values from describe-certificate
- If CAA records exist, ensure they include `amazon.com` or `amazontrust.com`: add CAA record `0 issue "amazon.com"`

SHOULD:
- Wait for DNS propagation (up to 30 minutes for Route 53, longer for external DNS)
- Use `aws acm describe-certificate` to monitor validation status after CNAME creation
- If validation timed out (72 hours), delete the certificate and request a new one

MAY:
- Automate CNAME creation using `aws acm` with Route 53 integration in CloudFormation or Terraform

## Common Issues

- symptoms: "CNAME record exists but validation still pending"
  diagnosis: "CNAME values may be incorrect or DNS has not propagated."
  resolution: "Compare CNAME Name/Value exactly with describe-certificate output. Check propagation with dig."

- symptoms: "Validation fails for wildcard domain"
  diagnosis: "Wildcard domains (*.example.com) require their own CNAME validation record, separate from the apex."
  resolution: "Create the CNAME for the wildcard domain as shown in DomainValidationOptions."

- symptoms: "CAA record blocking certificate issuance"
  diagnosis: "CAA records restrict which CAs can issue certificates for the domain."
  resolution: "Add CAA record: 0 issue \"amazon.com\" to allow ACM to issue certificates."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create/update CNAME validation record | GREEN | DNS record creation is non-destructive and reversible |
| Add CAA record for amazon.com | GREEN | Additive DNS change, does not affect existing certificates |
| Delete timed-out certificate and re-request | YELLOW | Deleting a FAILED certificate is safe, but new request generates different CNAME values |

## Escalation Conditions

- Certificate is in use by production services and validation is failing on renewal — escalate to certificate owner and service team
- CAA record changes affect multiple domains or certificates — escalate to DNS administrator
- Repeated validation failures with correct CNAME records — escalate to AWS Support (possible service issue)
- Certificate deletion required for a certificate with InUseBy associations — escalate to service owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, validation records, and associated service ARNs |
| `aws route53 list-resource-record-sets` | MEDIUM | Exposes DNS zone configuration and record values |
| `dig CAA / dig CNAME` | LOW | Public DNS query results |
| `aws cloudtrail lookup-events` | MEDIUM | May contain IAM principal and request details |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services (check InUseBy first)
- **NEVER** suggest using self-signed certificates in production as a workaround for validation failures
- **NEVER** suggest disabling certificate transparency logging as a troubleshooting step
- **NEVER** remove existing CAA records without understanding the full impact on other certificates

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created CNAME validation record | Delete the CNAME record from the hosted zone: `aws route53 change-resource-record-sets` with Action DELETE |
| Added CAA record | Remove the CAA record if it was not previously present |
| Deleted failed certificate | Cannot undo — re-request a new certificate if needed |
| Updated CNAME values | Restore original CNAME values from backup or previous describe-certificate output |

## Output Format

```yaml
root_cause: "dns_validation_failure — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<describe-certificate output>"
  - type: dns_records
    content: "<CNAME record status>"
severity: HIGH
mitigation:
  immediate: "Create or fix the DNS CNAME validation record"
  long_term: "Use CloudFormation/Terraform to automate certificate provisioning with DNS validation"
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
