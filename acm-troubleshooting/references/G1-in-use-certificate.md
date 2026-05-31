---
title: "G1 — In-Use Certificate Deletion"
description: "Diagnose and resolve errors when attempting to delete a certificate that is associated with AWS services"
status: active
severity: MEDIUM
triggers:
  - "cannot delete certificate"
  - "certificate in use"
  - "ResourceInUseException"
  - "delete certificate failed"
  - "disassociate certificate"
owner: devops-agent
objective: "Identify all services using the certificate and disassociate before deletion"
context: "ACM prevents deletion of certificates that are associated with AWS services (CloudFront, ELB, API Gateway, Elastic Beanstalk, etc.). The InUseBy field in describe-certificate lists all associated service ARNs. All associations must be removed before the certificate can be deleted."
---

## Phase 1 — Triage

MUST:
- Check which services use the certificate: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.InUseBy'`
- Identify each service ARN in the InUseBy list
- Verify the certificate you want to delete is the correct one: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{DomainName:DomainName,SANs:SubjectAlternativeNames,NotAfter:NotAfter}'`

SHOULD:
- Check if a replacement certificate is ready before disassociating
- For each service in InUseBy, determine the specific resource (distribution, listener, domain name)
- Plan the disassociation order to minimize downtime

MAY:
- Check if the certificate is referenced in CloudFormation stacks: `aws cloudformation describe-stack-resources --stack-name <stack> --query "StackResources[?ResourceType=='AWS::CertificateManager::Certificate']"`
- Verify no automation or IaC will re-associate the certificate after removal

## Phase 2 — Remediate

MUST:
- Disassociate from each service before deletion:
  - CloudFront: update distribution to use a different certificate or CloudFront default cert
  - ALB/NLB: remove from listener: `aws elbv2 remove-listener-certificates --listener-arn <listener-arn> --certificates CertificateArn=<cert-arn>` or modify the default certificate
  - API Gateway: update or delete the custom domain name
  - Elastic Beanstalk: update environment configuration
- After all disassociations: `aws acm delete-certificate --certificate-arn <cert-arn>`

SHOULD:
- Replace with a new certificate BEFORE removing the old one to avoid downtime
- For ALB with SNI: add the new certificate first, then remove the old one
- Verify deletion succeeded: `aws acm describe-certificate --certificate-arn <cert-arn>` should return error

MAY:
- Clean up DNS validation CNAME records for the deleted certificate
- Update IaC templates to reference the new certificate ARN
- Document the certificate replacement in change management

## Common Issues

- symptoms: "ResourceInUseException when calling DeleteCertificate"
  diagnosis: "Certificate is still associated with one or more AWS services."
  resolution: "Check InUseBy field and disassociate from all services first."

- symptoms: "InUseBy shows a service that no longer exists"
  diagnosis: "Stale association — the service was deleted but the association was not cleaned up."
  resolution: "Contact AWS Support to remove the stale association, or wait for it to clear."

- symptoms: "Certificate associated with CloudFormation-managed resource"
  diagnosis: "Deleting the certificate outside CloudFormation will cause stack drift."
  resolution: "Update the CloudFormation stack to use a different certificate, then delete the old one."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Disassociate certificate from CloudFront | RED | Causes HTTPS to fail if no replacement certificate |
| Remove certificate from ALB/NLB listener | RED | Removes TLS termination for that domain |
| Update API Gateway custom domain | YELLOW | May cause brief API unavailability |
| Delete certificate after disassociation | YELLOW | Permanent deletion; cannot be undone |

## Escalation Conditions

- Certificate deletion required for a production service — escalate to service owner for coordinated replacement
- Stale InUseBy association (service no longer exists) — escalate to AWS Support
- Certificate managed by CloudFormation — escalate to IaC team to avoid stack drift
- Multiple services using the same certificate — escalate to infrastructure team for coordinated migration

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm describe-certificate` (InUseBy) | MEDIUM | Exposes service ARNs associated with the certificate |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names and certificate details |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services without first disassociating and replacing
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** remove a certificate from a service without having a replacement ready
- **NEVER** force-delete a certificate by removing service associations without coordinating with service owners

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Disassociated certificate from service | Re-associate the certificate with the service |
| Deleted certificate | Cannot undo — re-request or re-import if needed |
| Updated CloudFormation stack | Revert CloudFormation stack to previous template |

## Output Format

```yaml
root_cause: "in_use_certificate — <specific_cause>"
evidence:
  - type: in_use_by
    content: "<list of service ARNs using the certificate>"
  - type: certificate_details
    content: "<certificate domain and expiry>"
severity: MEDIUM
mitigation:
  immediate: "Disassociate certificate from all services, then delete"
  long_term: "Use IaC to manage certificate lifecycle and associations"
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
