---
title: "C1 — COPY Failures"
description: "Diagnose and resolve COPY command failures in Redshift"
status: active
severity: HIGH
triggers:
  - "COPY failed"
  - "COPY error"
  - "stl_load_errors"
  - "S3ServiceException"
  - "data load failed"
  - "load error"
owner: devops-agent
objective: "Identify the root cause of COPY command failures and fix data loading"
context: "COPY is the primary method for bulk loading data into Redshift from S3, DynamoDB, EMR, or remote hosts. Failures can be caused by S3 access issues, data format mismatches, column type errors, manifest file problems, or IAM role configuration."
---

## Phase 1 — Triage

MUST:
- Check COPY errors: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, filename, line_number, colname, type, raw_field_value, err_reason FROM stl_load_errors ORDER BY starttime DESC LIMIT 20"`
- Check COPY command history: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, filename, lines_scanned, status FROM stl_load_commits ORDER BY starttime DESC LIMIT 10"`
- Verify IAM role attached to cluster: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].IamRoles'`
- Test S3 access: `aws s3 ls <s3-path> --profile <role-profile>`

SHOULD:
- Check if MAXERROR threshold was exceeded
- Verify file format matches COPY options (CSV, JSON, Parquet, delimiter)
- Check for encoding issues (UTF-8 BOM, special characters)
- Verify manifest file if using MANIFEST option

MAY:
- Check S3 bucket policy for cross-account access
- Verify KMS key permissions if S3 objects are encrypted
- Check for S3 eventual consistency issues with recently written files

## Phase 2 — Remediate

MUST:
- If S3 access denied: verify IAM role has s3:GetObject and s3:ListBucket on the bucket/prefix
- If data type mismatch: fix source data or adjust column types
- If delimiter issues: specify correct DELIMITER, QUOTE, or use FORMAT AS options
- If manifest errors: validate manifest JSON format and S3 paths

SHOULD:
- Use MAXERROR to allow some bad rows and review stl_load_errors
- Use NOLOAD option to validate data without actually loading
- Specify REGION if S3 bucket is in a different region than the cluster

MAY:
- Use COPY with COMPUPDATE OFF STATUPDATE OFF for faster initial loads
- Split large files into multiple 100MB-1GB files for parallel loading
- Use gzip or lzo compression for faster S3 transfer

## Common Issues

- symptoms: "S3ServiceException: Access Denied"
  diagnosis: "IAM role lacks S3 permissions or bucket policy denies access."
  resolution: "Add s3:GetObject and s3:ListBucket to the IAM role. Check bucket policy. Verify role ARN in COPY command."

- symptoms: "Delimiter not found — extra columns in data"
  diagnosis: "Data contains the delimiter character within field values."
  resolution: "Use QUOTE option for CSV, or switch to a different delimiter. Consider Parquet format."

- symptoms: "Invalid digit, numeric overflow, or date format error"
  diagnosis: "Source data doesn't match target column type."
  resolution: "Check stl_load_errors for specific column and value. Fix source data or ALTER column type."

- symptoms: "COPY loads 0 rows with no errors"
  diagnosis: "S3 path is empty, manifest points to non-existent files, or REGION mismatch."
  resolution: "Verify S3 path has files. Check manifest entries. Add REGION parameter if cross-region."

## Output Format

```yaml
root_cause: "COPY failure — <specific_cause>"
evidence:
  - type: system_table
    content: "<stl_load_errors details>"
  - type: iam_role
    content: "<role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the identified permission, format, or data type issue"
  long_term: "Implement data validation before COPY, use NOLOAD for pre-checks, monitor stl_load_errors"
```


## Safety Ratings
```
safety_ratings:
  - "Query stl_load_errors: GREEN — read-only system table query"
  - "Check IAM role permissions: GREEN — read-only IAM inspection"
  - "Test S3 access: GREEN — read-only S3 listing"
  - "Use NOLOAD to validate data: GREEN — validates without loading"
  - "Fix IAM role permissions: YELLOW — changes access scope"
  - "ALTER column type: RED — may require table rebuild, affects existing data"
  - "Run COPY with MAXERROR: YELLOW — loads data while skipping bad rows"
```

## Escalation Conditions
- Cluster serves production analytics
- COPY failure blocking production data pipeline
- Fix requires IAM role changes affecting multiple services
- Data format issues requiring source system changes
- Cross-account S3 access requiring bucket policy changes

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "stl_load_errors raw_field_value: contains actual data values that failed to load"
    - "S3 paths: data source locations"
    - "IAM role ARNs: permission identifiers"
    - "User credentials: database usernames"
  handling: "stl_load_errors may contain sensitive data values. Mask raw_field_value in shared reports."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER use COPY with MAXERROR set to a very high value without reviewing the skipped rows
- NEVER grant s3:* permissions — use specific s3:GetObject and s3:ListBucket
- NEVER expose stl_load_errors raw_field_value externally (may contain PII)

## Phase 3 — Rollback
- If COPY loaded incorrect data: DELETE the loaded rows using a timestamp or batch identifier, then re-COPY with correct data
- If IAM role permissions were changed: revert to previous IAM policy version
- If column type was altered: ALTER back to original type (may require data migration)
- If MAXERROR was used: review stl_load_errors for skipped rows and reload them after fixing

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
