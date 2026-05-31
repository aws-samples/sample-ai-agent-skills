---
title: "E1 — CUR Setup"
description: "Diagnose Cost and Usage Report setup issues"
status: active
severity: MEDIUM
triggers:
  - "CUR not created"
  - "cost usage report setup"
  - "CUR configuration"
  - "billing report setup"
owner: devops-agent
objective: "Resolve CUR report creation and configuration issues"
context: "CUR delivers detailed billing data to S3. Reports can include resource IDs, cost allocation tags, and Savings Plans/RI details. First delivery takes up to 24 hours. S3 bucket must have correct policy."
---

## Phase 1 — Triage

MUST:
- Check existing report definitions: `aws cur describe-report-definitions`
- Verify S3 bucket exists and has correct policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Check if report was recently created (first delivery takes up to 24 hours)
- Verify report region (CUR API is only available in us-east-1)

SHOULD:
- Check S3 bucket for report files: `aws s3 ls s3://<bucket>/<prefix>/`
- Verify report includes desired columns (resource IDs, tags)
- Check report format (CSV, Parquet) and compression

MAY:
- Verify Athena integration if configured
- Check if report refresh is enabled for updated data

## Phase 2 — Remediate

MUST:
- Create CUR report (must use us-east-1 region for API): `aws cur put-report-definition --report-definition '{"ReportName":"<name>","TimeUnit":"DAILY","Format":"Parquet","Compression":"Parquet","S3Bucket":"<bucket>","S3Prefix":"<prefix>","S3Region":"<region>","AdditionalSchemaElements":["RESOURCES"]}'`
- Add S3 bucket policy for billingreports.amazonaws.com
- Wait up to 24 hours for first delivery

SHOULD:
- Include RESOURCES schema element for resource-level detail
- Use Parquet format for efficient querying
- Enable report refresh for updated data

MAY:
- Set up Athena integration for SQL querying
- Configure QuickSight for visualization

## Common Issues

- symptoms: "CUR report not appearing in S3"
  diagnosis: "Report recently created (takes up to 24 hours) or S3 bucket policy missing."
  resolution: "Wait 24 hours. Verify bucket policy grants billingreports.amazonaws.com access."

- symptoms: "CUR API returns empty results"
  diagnosis: "CUR API only works in us-east-1 region."
  resolution: "Use us-east-1 endpoint: `aws cur describe-report-definitions --region us-east-1`"

- symptoms: "Report missing resource IDs"
  diagnosis: "RESOURCES schema element not included in report definition."
  resolution: "Recreate report with AdditionalSchemaElements including RESOURCES."

## Output Format

```yaml
root_cause: "cur_setup — <specific_cause>"
evidence:
  - type: report_definition
    content: "<CUR report configuration>"
  - type: s3_bucket
    content: "<S3 bucket policy and contents>"
severity: MEDIUM
mitigation:
  immediate: "Fix report definition, bucket policy, or wait for delivery"
  long_term: "Set up Athena integration for automated cost analysis"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — CUR contains detailed billing data including resource IDs and usage |
| Blast Radius | LOW — CUR setup affects reporting only, not infrastructure |
| Rollback Complexity | LOW — Reports can be deleted and recreated |
| Compliance Scope | HIGH — CUR data supports financial auditing and compliance |

## Escalation Conditions

- Escalate to security if CUR S3 bucket lacks encryption or has overly permissive access
- Escalate to finance if CUR delivery failures create gaps in billing records
- Escalate to compliance if CUR data is needed for audit deadlines

## Data Sensitivity

- **CUR data** contains detailed billing including resource IDs, usage, and costs — highly sensitive
- **S3 bucket** storing CUR must have encryption, versioning, and strict access controls
- **Athena queries** on CUR data may expose financial details — restrict access

## PROHIBITED Actions

- Do NOT store CUR data in unencrypted S3 buckets
- Do NOT grant public access to CUR S3 buckets
- Do NOT delete CUR data without confirming retention requirements are met
- Do NOT share CUR data outside authorized finance and operations teams

## Phase 3 — Rollback

- If report was misconfigured, delete and recreate with correct settings
- Restore S3 bucket policy if it was modified during troubleshooting
- Verify report delivery resumes within 24 hours of fix
- Document any gaps in CUR data for audit purposes

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
