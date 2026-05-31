---
title: "E2 — S3 Integration"
description: "Diagnose S3-based data integration issues with Supply Chain"
status: active
severity: MEDIUM
triggers: ["S3 integration error", "cannot read S3", "S3 bucket access", "file format error"]
owner: devops-agent
objective: "Resolve S3-based data integration issues in Supply Chain"
context: "S3 is the primary data exchange mechanism. File format, folder structure, and permissions must match Supply Chain expectations."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket exists and is accessible: `aws s3 ls s3://<bucket-name>/`
- Check bucket policy: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify file format and folder structure match expectations

SHOULD:
- Check IAM role permissions for S3 access
- Verify bucket and instance are in the same region

MAY:
- Check S3 access logs for denied requests
- Review file encoding and delimiter settings

## Phase 2 — Remediate

MUST:
- Fix S3 bucket policy to allow Supply Chain access
- Fix IAM role permissions: s3:GetObject, s3:ListBucket
- Ensure correct file format (CSV, JSON, or Parquet)

SHOULD:
- Standardize file naming conventions
- Implement S3 event notifications for automated processing

MAY:
- Use S3 lifecycle policies for data retention
- Implement S3 versioning for data recovery

## Safety Ratings

```
safety_ratings:
  - "s3 ls: GREEN — read-only bucket listing"
  - "s3api get-bucket-policy: GREEN — read-only policy check"
  - "Bucket policy modification: YELLOW — changes access permissions"
  - "IAM role modification: YELLOW — changes service access"
```

## Escalation Conditions

- Cross-account S3 access requires organizational policy changes
- S3 bucket encryption conflicts with Supply Chain requirements
- Large data volumes cause S3 throttling
- File format changes require upstream system modifications

## Data Sensitivity

- S3 bucket contents: HIGH — contains supply chain data
- Bucket policy: MEDIUM — reveals access configuration
- File contents: HIGH — proprietary business data
- S3 access logs: MEDIUM — reveals access patterns

## Prohibited Actions

- NEVER make S3 buckets public for troubleshooting
- NEVER disable S3 encryption on buckets containing business data
- NEVER delete S3 data without confirming it's been processed
- NEVER use overly permissive bucket policies (s3:*)

## Phase 3 — Rollback

1. If bucket policy was modified: restore original policy
2. If IAM role permissions were broadened: revert to least-privilege
3. If files were moved or renamed: restore original structure
4. Verify data integration flows work after S3 changes
5. Document all S3 configuration changes

## Common Issues

- symptoms: "Data integration flow cannot read from S3 bucket"
  diagnosis: "IAM role lacks s3:GetObject permission or bucket policy blocks access."
  resolution: "Update IAM role with s3:GetObject and s3:ListBucket. Check bucket policy allows the role."

- symptoms: "Files in S3 are not picked up by integration flow"
  diagnosis: "Files are in wrong folder path or have incorrect naming convention."
  resolution: "Verify file path matches flow configuration. Check file extension and naming pattern."

## Output Format

```yaml
root_cause: "s3_integration — <specific_cause>"
evidence:
  - type: s3_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 access and file format"
  long_term: "Standardize S3 data exchange patterns"
```

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
