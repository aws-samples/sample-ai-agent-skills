---
title: "C2 — UNLOAD Errors"
description: "Diagnose and resolve UNLOAD command failures in Redshift"
status: active
severity: MEDIUM
triggers:
  - "UNLOAD failed"
  - "UNLOAD error"
  - "export to S3"
  - "cannot write to S3"
owner: devops-agent
objective: "Identify and fix UNLOAD command failures for S3 data export"
context: "UNLOAD exports query results from Redshift to S3 in parallel across compute nodes. Failures typically involve S3 write permissions, KMS encryption, existing file conflicts, or query errors."
---

## Phase 1 — Triage

MUST:
- Check UNLOAD errors in query log: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, substring, elapsed FROM svl_qlog WHERE substring LIKE '%UNLOAD%' ORDER BY starttime DESC LIMIT 10"`
- Verify IAM role has S3 write permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:PutObject s3:GetBucketLocation --resource-arns arn:aws:s3:::<bucket>/*`
- Check if target S3 path already has files (UNLOAD fails if files exist without ALLOWOVERWRITE): `aws s3 ls <s3-path>/`
- Verify IAM role is attached to cluster: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].IamRoles[*].IamRoleArn'`

SHOULD:
- Check KMS key permissions if using encrypted UNLOAD
- Verify S3 bucket exists and is in expected region
- Check for S3 bucket policy restrictions

MAY:
- Check UNLOAD query for syntax errors
- Verify PARALLEL ON/OFF setting matches expectations
- Check for S3 VPC endpoint if cluster is in VPC

## Phase 2 — Remediate

MUST:
- If S3 write denied: add s3:PutObject permission to IAM role for the target bucket/prefix
- If files already exist: add ALLOWOVERWRITE option or use a unique prefix
- If KMS error: grant kms:GenerateDataKey and kms:Encrypt to the IAM role
- If query error: fix the SELECT query used in UNLOAD

SHOULD:
- Use PARALLEL ON (default) for faster export
- Use Parquet format for downstream analytics: `UNLOAD ('SELECT ...') TO 's3://...' IAM_ROLE '...' FORMAT AS PARQUET`
- Add MANIFEST option to generate a manifest file listing all output files

MAY:
- Use MAXFILESIZE to control output file sizes
- Add GZIP or ZSTD compression for smaller output
- Use PARTITION BY for partitioned output

## Common Issues

- symptoms: "S3ServiceException: Access Denied on PutObject"
  diagnosis: "IAM role lacks s3:PutObject on the target bucket/prefix."
  resolution: "Add s3:PutObject and s3:GetBucketLocation to the IAM role policy."

- symptoms: "UNLOAD fails with 'already exists' error"
  diagnosis: "Target S3 prefix already contains files from a previous UNLOAD."
  resolution: "Add ALLOWOVERWRITE option or use a unique prefix with timestamp."

## Output Format

```yaml
root_cause: "UNLOAD error — <specific_cause>"
evidence:
  - type: system_table
    content: "<svl_qlog UNLOAD error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 permissions, add ALLOWOVERWRITE, or correct query"
  long_term: "Automate UNLOAD with unique prefixes, monitor IAM role permissions"
```


## Safety Ratings
```
safety_ratings:
  - "Query svl_qlog for UNLOAD errors: GREEN — read-only system table query"
  - "Check IAM role permissions: GREEN — read-only IAM inspection"
  - "Check S3 path for existing files: GREEN — read-only S3 listing"
  - "Fix IAM role S3 write permissions: YELLOW — changes access scope"
  - "Use ALLOWOVERWRITE: YELLOW — overwrites existing S3 files permanently"
  - "Run UNLOAD with corrected parameters: GREEN — exports data to S3"
```

## Escalation Conditions
- Cluster serves production analytics
- UNLOAD failure blocking data export pipeline
- Fix requires IAM role changes or S3 bucket policy updates
- KMS encryption issues requiring key policy changes
- Cross-account S3 write access needed

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "UNLOAD query results: exported business data in S3"
    - "S3 paths: data export destinations"
    - "IAM role ARNs: permission identifiers"
  handling: "UNLOAD exports query results to S3 — ensure destination bucket has appropriate encryption and access controls."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER UNLOAD sensitive data to unencrypted or publicly accessible S3 buckets
- NEVER use ALLOWOVERWRITE without confirming the existing files are not needed
- NEVER grant s3:* permissions — use specific s3:PutObject and s3:GetBucketLocation

## Phase 3 — Rollback
- If ALLOWOVERWRITE was used: previous S3 files are permanently overwritten — restore from S3 versioning if enabled
- If IAM role permissions were changed: revert to previous IAM policy version
- If KMS key policy was modified: restore previous key policy
- If UNLOAD exported incorrect data: delete the S3 files and re-run UNLOAD with correct query

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
