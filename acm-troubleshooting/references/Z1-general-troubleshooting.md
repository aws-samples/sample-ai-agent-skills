---
title: "Z1 — General ACM Troubleshooting (Catch-All)"
description: "Fallback SOP for ACM issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown ACM issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all certificates and their statuses: `aws acm list-certificates --certificate-statuses ISSUED PENDING_VALIDATION FAILED EXPIRED REVOKED INACTIVE`
- If a specific certificate is involved, get full details: `aws acm describe-certificate --certificate-arn <cert-arn>`
- Check certificate status, validation method, expiry, and associations
- Check for recent ACM events in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=acm.amazonaws.com --max-results 10`

SHOULD:
- Check private CAs if applicable: `aws acm-pca list-certificate-authorities`
- Check CloudWatch DaysToExpiry for certificates approaching expiry
- Verify IAM permissions for ACM operations
- Check the region — ensure you are looking in the correct region for the target service

MAY:
- Check AWS Health Dashboard for ACM service events
- Review AWS Config for certificate compliance rules
- Check Service Quotas for ACM limits: `aws service-quotas get-service-quota --service-code acm --quota-code L-F141DD1D`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- DNS validation issues → Validation (A1)
- Email validation issues → Validation (A2)
- Certificate stuck pending → Validation (A3)
- Auto-renewal failures → Renewal (B1)
- Manual renewal needed → Renewal (B2)
- Imported cert expiring → Renewal (B3)
- CloudFront certificate errors → Integration (C1)
- ALB/NLB certificate issues → Integration (C2)
- API Gateway custom domain → Integration (C3)
- Certificate not visible in service → Integration (C4)
- Private CA problems → Private CA (D1)
- Private cert issuance failures → Private CA (D2)
- Certificate import errors → Import (E1)
- Chain of trust issues → Import (E2)
- Key algorithm incompatibility → Security (F1)
- Certificate transparency concerns → Security (F2)
- Cannot delete certificate → Deletion (G1)
- Unused certificate cleanup → Deletion (G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all certificate configurations systematically
- Review CloudTrail for recent ACM configuration changes
- Verify DNS configuration for all certificate domains
- Check associated AWS services for certificate-related errors

SHOULD:
- Check AWS Health Dashboard for ACM service events
- Compare with a known-good certificate configuration
- Test SSL/TLS connectivity: `openssl s_client -connect <domain>:443 -servername <domain>`

MAY:
- Check Certificate Transparency logs for the domain: search crt.sh
- Review CAA records for the domain
- Check for regional service outages

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Classify and pivot to specific SOP | GREEN | Investigation only, no state changes |
| Check certificate configurations | GREEN | Read-only operations |
| Test SSL/TLS connectivity | GREEN | Read-only diagnostic |

## Escalation Conditions

- Issue cannot be classified after systematic triage — escalate to AWS Support
- Multiple concurrent certificate issues affecting production — escalate to incident management
- Security-related certificate issue (unauthorized issuance, compromise) — escalate immediately to security team
- Issue spans multiple AWS services or accounts — escalate to infrastructure team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws acm list-certificates` | MEDIUM | Lists all certificate ARNs and domains |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names, status, and service associations |
| `aws cloudtrail lookup-events` | MEDIUM | May contain IAM principal and request details |
| `aws acm-pca list-certificate-authorities` | MEDIUM | Exposes private CA configuration |
| `openssl s_client -connect` | MEDIUM | Exposes server certificate and chain |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** make state-changing remediations during general troubleshooting without first classifying the issue

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 triage |
| Pivot to specific SOP | Follow rollback steps in the specific SOP |

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<validation|renewal|integration|private_ca|import|security|deletion|unknown>"
investigation_path: "certificate status → validation → associations → <domain_classification>"
evidence:
  - type: certificate_config
    content: "<certificate configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
