---
title: "C2 — Sensitive Data Discovery"
description: "Diagnose sensitive data discovery issues in S3 buckets"
status: active
severity: HIGH
triggers:
  - "sensitive data not found"
  - "discovery not working"
  - "no findings for bucket"
  - "scanning incomplete"
owner: devops-agent
objective: "Ensure sensitive data in S3 buckets is properly discovered by Macie"
context: "Sensitive data discovery requires active discovery jobs targeting specific buckets. Common issues include no job configured for the bucket, unsupported file formats, KMS access issues, or sampling missing the sensitive objects."
---

## Phase 1 — Triage

MUST:
- Check if a discovery job covers the bucket: `aws macie2 list-classification-jobs --filter-criteria '{"includes":{"key":"bucketDefinitions.bucketName","values":["<bucket-name>"]}}'`
- Check job results: `aws macie2 describe-classification-job --job-id <job-id>`
- Verify bucket accessibility: `aws macie2 describe-buckets --criteria '{"bucketName":{"eq":["<bucket-name>"]}}'`
- Check if automated sensitive data discovery is enabled: `aws macie2 get-automated-discovery-configuration`

SHOULD:
- Verify file formats in the bucket are supported by Macie
- Check sampling depth configuration on the job
- Review managed and custom identifiers enabled for the job

MAY:
- Check if objects are encrypted with a KMS key Macie cannot access
- Verify objects are not in Glacier storage class

## Phase 2 — Remediate

MUST:
- Create a discovery job for the bucket if none exists
- Grant Macie access to KMS keys used for bucket encryption
- Ensure job includes appropriate managed and custom identifiers

SHOULD:
- Use 100% sampling for compliance-critical buckets
- Enable automated sensitive data discovery for ongoing monitoring
- Configure job to scan all supported file types

MAY:
- Create targeted jobs for specific prefixes containing sensitive data
- Set up scheduled jobs for regular scanning

## Common Issues

- symptoms: "No findings for a bucket known to contain PII"
  diagnosis: "No discovery job configured for the bucket."
  resolution: "Create a discovery job targeting the bucket."

- symptoms: "Job completed but missed known sensitive files"
  diagnosis: "Files in unsupported format or sampling missed them."
  resolution: "Check file formats. Increase sampling depth to 100%."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create discovery job | GREEN | New scan job, non-destructive |
| Grant Macie KMS key access | YELLOW | Key policy changes affect all key usage |
| Enable automated discovery | YELLOW | Ongoing scanning with cost implications |
| Increase sampling to 100% | YELLOW | Significant cost increase for large buckets |

## Escalation Conditions

- Known sensitive data not being discovered — escalate to security team
- Discovery job access denied on compliance-critical buckets — escalate to bucket owner
- KMS key policy changes required — escalate to key administrator
- Automated discovery cost concerns — escalate to cost management team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 list-classification-jobs` | HIGH | Exposes job configurations and target buckets |
| `aws macie2 describe-classification-job` | HIGH | Exposes detailed job configuration |
| `aws macie2 describe-buckets` | HIGH | Exposes bucket metadata |
| `aws macie2 get-automated-discovery-configuration` | MEDIUM | Exposes automated discovery settings |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** create discovery jobs on buckets without authorization from the bucket owner
- **NEVER** enable 100% sampling on large buckets without cost analysis

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created discovery job | Cancel: `aws macie2 update-classification-job --job-id <id> --job-status CANCELLED` |
| Granted KMS key access | Remove Macie from KMS key policy |
| Enabled automated discovery | Disable automated discovery |
| Increased sampling | Reduce sampling depth in job configuration |

## Output Format

```yaml
root_cause: "sensitive_data_discovery — <specific_cause>"
evidence:
  - type: job_coverage
    content: "<discovery jobs covering the bucket>"
  - type: bucket_status
    content: "<bucket accessibility and metadata>"
severity: HIGH
mitigation:
  immediate: "Create or fix discovery job for the bucket"
  long_term: "Ensure all sensitive buckets have scheduled discovery jobs"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
