---
title: "F1 — Report Generation"
description: "Diagnose assessment report generation failures"
status: active
severity: HIGH
triggers: ["report failed", "cannot generate report", "report error"]
owner: devops-agent
objective: "Resolve report generation issues"
context: "Assessment reports are generated as PDF/CSV and stored in S3. Generation requires S3 bucket access and sufficient evidence."
---

## Phase 1 — Triage

MUST:
- Check S3 bucket configuration: `aws auditmanager get-settings --attribute DEFAULT_ASSESSMENT_REPORTS_DESTINATION`
- Verify S3 bucket exists and is accessible
- Check IAM permissions for report generation

SHOULD:
- Verify assessment has evidence to report on
- Check S3 bucket policy allows Audit Manager writes

MAY:
- Review CloudTrail for report generation errors

## Phase 2 — Remediate

MUST:
- Configure valid S3 bucket in Audit Manager settings
- Ensure bucket policy allows auditmanager.amazonaws.com to write
- Fix IAM permissions

SHOULD:
- Verify bucket encryption settings are compatible

MAY:
- Use separate bucket for reports with appropriate retention

## Common Issues

- symptoms: "Report generation fails with S3 access error"
  diagnosis: "S3 bucket policy doesn't allow Audit Manager writes."
  resolution: "Add s3:PutObject permission for auditmanager.amazonaws.com to bucket policy."

## Output Format

```yaml
root_cause: "report_generation — <specific_cause>"
evidence:
  - type: report_error
    content: "<generation error details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 access"
  long_term: "Validate S3 configuration during setup"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Configure valid S3 bucket in Audit Manager settings | YELLOW | Changes report storage location |
| Ensure bucket policy allows auditmanager writes | YELLOW | S3 policy change — affects report delivery |
| Fix IAM permissions | YELLOW | IAM policy modification |
| Verify bucket encryption settings | GREEN | Verification only |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Report storage location changes affect audit evidence chain of custody
- S3 bucket policy changes affect report security

## Data Sensitivity

- Assessment reports contain detailed compliance findings and evidence summaries
- S3 bucket stores sensitive compliance documentation
- Report generation reveals compliance posture across assessed resources

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER store compliance reports in unencrypted S3 buckets
- NEVER grant public access to report storage buckets

## Phase 3 — Rollback

- If S3 bucket configuration was changed: restore original bucket settings
- If bucket policy was modified: revert to original policy
- If IAM permissions were changed: revert to previous IAM policy
- Document all report configuration changes for compliance audit trail

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
