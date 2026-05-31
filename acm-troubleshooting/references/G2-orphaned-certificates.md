---
title: "G2 — Orphaned Certificates"
description: "Identify and clean up unused or orphaned ACM certificates"
status: active
severity: LOW
triggers:
  - "orphaned certificate"
  - "unused certificate"
  - "certificate cleanup"
  - "certificate inventory"
  - "too many certificates"
owner: devops-agent
objective: "Identify certificates that are no longer in use and safely clean them up"
context: "Over time, ACM accounts accumulate certificates that are no longer associated with any service — from decommissioned services, failed validations, test certificates, or replaced certificates. While unused certificates don't incur direct costs (public ACM certs are free), they create management overhead and potential confusion."
---

## Phase 1 — Triage

MUST:
- List all certificates: `aws acm list-certificates --certificate-statuses ISSUED EXPIRED FAILED INACTIVE PENDING_VALIDATION REVOKED`
- For each certificate, check if it's in use: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{DomainName:DomainName,Status:Status,InUseBy:InUseBy,NotAfter:NotAfter,Type:Type}'`
- Identify certificates with empty InUseBy arrays (not associated with any service)
- Identify certificates in FAILED, EXPIRED, or REVOKED status

SHOULD:
- Check all regions for orphaned certificates (certificates are regional)
- Script the inventory: `for arn in $(aws acm list-certificates --query 'CertificateSummaryList[].CertificateArn' --output text); do echo "---"; aws acm describe-certificate --certificate-arn $arn --query 'Certificate.{Arn:CertificateArn,Domain:DomainName,Status:Status,InUseBy:InUseBy,Expiry:NotAfter}'; done`
- Check if orphaned certificates are referenced in IaC templates (CloudFormation, Terraform)
- Verify certificates are not used by services outside ACM's tracking (e.g., manual EC2 configurations)

MAY:
- Use AWS Config to track certificate resources across the organization
- Check CloudTrail for recent usage of the certificate ARN
- Export a certificate inventory report for review

## Phase 2 — Remediate

MUST:
- Delete FAILED certificates (cannot be used): `aws acm delete-certificate --certificate-arn <cert-arn>`
- Delete EXPIRED certificates that are not in use
- Confirm with the team before deleting ISSUED certificates with empty InUseBy

SHOULD:
- Delete certificates in batches, verifying each is truly unused
- Clean up associated DNS validation CNAME records for deleted certificates
- Remove references from IaC templates before deleting

MAY:
- Set up a periodic cleanup process (monthly or quarterly)
- Tag certificates with owner and purpose for easier management: `aws acm add-tags-to-certificate --certificate-arn <cert-arn> --tags Key=Owner,Value=<team> Key=Purpose,Value=<purpose>`
- Implement a certificate governance policy

## Common Issues

- symptoms: "Hundreds of certificates in the account"
  diagnosis: "Accumulated from CI/CD pipelines, testing, or lack of cleanup process."
  resolution: "Inventory all certificates, identify unused ones, and delete in batches."

- symptoms: "Certificate shows empty InUseBy but is actually in use"
  diagnosis: "Some services may not report back to ACM's InUseBy tracking."
  resolution: "Check CloudTrail for recent GetCertificate or DescribeCertificate calls referencing the ARN."

- symptoms: "Want to prevent certificate sprawl"
  diagnosis: "No governance process for certificate lifecycle."
  resolution: "Implement tagging policy, periodic audits, and IaC-managed certificates."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Delete FAILED certificates | GREEN | FAILED certificates cannot be used; safe to remove |
| Delete EXPIRED certificates (not in use) | GREEN | Expired certificates provide no value |
| Delete ISSUED certificates with empty InUseBy | YELLOW | Verify truly unused before deletion |
| Tag certificates for management | GREEN | Metadata-only change |

## Escalation Conditions

- ISSUED certificate with empty InUseBy but referenced in IaC templates — escalate to IaC team
- Large-scale cleanup (50+ certificates) — escalate to infrastructure team for review
- Certificates potentially used by services outside ACM tracking — escalate to service owners
- Compliance requirements for certificate retention — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm list-certificates` | MEDIUM | Lists all certificate ARNs and domains |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, status, and service associations |
| `aws cloudtrail lookup-events` | MEDIUM | May contain IAM principal and request details |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** bulk-delete certificates without individual verification of InUseBy status
- **NEVER** delete certificates referenced in active CloudFormation stacks or Terraform state

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Deleted FAILED/EXPIRED certificates | Cannot undo — re-request if needed |
| Deleted unused ISSUED certificates | Cannot undo — re-request or re-import if needed |
| Added tags to certificates | Remove tags: `aws acm remove-tags-from-certificate` |
| Removed DNS validation CNAME records | Recreate CNAME records if certificate is re-requested |

## Output Format

```yaml
root_cause: "orphaned_certificates — <specific_cause>"
evidence:
  - type: certificate_inventory
    content: "<list of orphaned certificates with status>"
  - type: cleanup_candidates
    content: "<certificates safe to delete>"
severity: LOW
mitigation:
  immediate: "Delete FAILED and EXPIRED certificates, review unused ISSUED certificates"
  long_term: "Implement certificate tagging, periodic audits, and IaC management"
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
