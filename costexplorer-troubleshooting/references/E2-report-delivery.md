---
title: "E2 — Report Delivery"
description: "Diagnose CUR report delivery failures"
status: active
severity: MEDIUM
triggers:
  - "CUR not delivered"
  - "report missing from S3"
  - "stale CUR data"
  - "report delivery failed"
owner: devops-agent
objective: "Resolve CUR report delivery issues"
context: "CUR reports are delivered to S3 multiple times per day as data finalizes. Delivery failures are caused by S3 bucket policy issues, bucket deletion, or region mismatches. Reports are updated incrementally for the current month."
---

## Phase 1 — Triage

MUST:
- Verify report definition exists: `aws cur describe-report-definitions --region us-east-1`
- Check S3 bucket exists: `aws s3api head-bucket --bucket <bucket>`
- Verify bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Check for recent report files: `aws s3 ls s3://<bucket>/<prefix>/ --recursive | tail -20`

SHOULD:
- Check if bucket region matches report S3Region setting
- Verify no S3 lifecycle rules are deleting reports prematurely
- Check CloudTrail for S3 PutObject failures from billing service

MAY:
- Verify report manifest file for latest delivery timestamp
- Check if bucket encryption is compatible with billing service

## Phase 2 — Remediate

MUST:
- Fix S3 bucket policy to allow billingreports.amazonaws.com
- Ensure bucket exists and is accessible
- Verify report definition S3Region matches bucket region

SHOULD:
- Check and fix S3 lifecycle rules if deleting reports too early
- Enable S3 versioning for report history
- Monitor report delivery with S3 event notifications

MAY:
- Set up automated report processing pipeline
- Configure alerts for missing report deliveries

## Common Issues

- symptoms: "No new report files for several days"
  diagnosis: "S3 bucket policy changed or bucket deleted."
  resolution: "Verify bucket exists and policy grants billing service access."

- symptoms: "Report data seems stale"
  diagnosis: "Report refresh not enabled or data still finalizing."
  resolution: "Enable report refresh. Current month data updates multiple times daily."

- symptoms: "Report files in wrong format"
  diagnosis: "Report definition format/compression settings incorrect."
  resolution: "Update report definition with desired format (CSV/Parquet)."

## Output Format

```yaml
root_cause: "report_delivery — <specific_cause>"
evidence:
  - type: report_config
    content: "<report definition and S3 settings>"
  - type: s3_contents
    content: "<recent report files in S3>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 bucket policy or report configuration"
  long_term: "Monitor report delivery and set up processing pipeline"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Report data contains detailed billing information |
| Blast Radius | LOW — Delivery issues affect reporting, not infrastructure |
| Rollback Complexity | LOW — Fix delivery configuration and wait for next delivery |
| Compliance Scope | HIGH — Gaps in report delivery affect financial audit trails |

## Escalation Conditions

- Escalate to finance if report delivery gaps exceed 48 hours
- Escalate to security if S3 bucket policy changes caused delivery failures
- Escalate to compliance if delivery gaps affect audit timelines

## Data Sensitivity

- **Report files** contain detailed billing data — highest financial sensitivity
- **S3 bucket** must maintain encryption and access controls
- **Lifecycle rules** must not delete reports before retention requirements are met

## PROHIBITED Actions

- Do NOT modify S3 bucket policies without verifying billing service access is maintained
- Do NOT set lifecycle rules that delete reports before the required retention period
- Do NOT disable S3 versioning on CUR buckets

## Phase 3 — Rollback

- Restore S3 bucket policy to allow billingreports.amazonaws.com access
- Restore lifecycle rules if they were incorrectly modified
- Wait for next delivery cycle (reports deliver multiple times daily)
- Document any delivery gaps for audit records

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
