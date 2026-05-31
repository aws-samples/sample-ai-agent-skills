---
title: "B1 — Import Job Failures"
description: "Diagnose DICOM import job failures"
status: active
severity: HIGH
triggers: ["import job failed", "DICOM import error", "start-dicom-import-job failure"]
owner: devops-agent
objective: "Resolve DICOM import job failures in HealthImaging"
context: "Import jobs read DICOM files from S3, validate them, and store in a datastore. Jobs can partially succeed."
---

## Phase 1 — Triage

MUST:
- Check import job status: `aws medical-imaging get-dicom-import-job --datastore-id <datastore-id> --job-id <job-id>`
- List recent import jobs: `aws medical-imaging list-dicom-import-jobs --datastore-id <datastore-id>`
- Verify S3 source bucket access: `aws s3 ls s3://<input-bucket>/<prefix>/`

SHOULD:
- Check the output manifest in S3 for per-file error details
- Verify IAM role used by the import job has S3 read and HealthImaging write permissions

MAY:
- Review CloudTrail for StartDICOMImportJob events
- Check S3 bucket policy for cross-account access issues

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions: s3:GetObject on source bucket, medical-imaging:* on datastore
- Ensure S3 bucket and datastore are in the same region
- Fix DICOM file issues identified in the output manifest

SHOULD:
- Retry failed imports after fixing file issues
- Use a dedicated IAM role for import jobs with least-privilege permissions

MAY:
- Implement pre-validation of DICOM files before import
- Set up S3 event notifications to trigger imports automatically

## Safety Ratings

```
safety_ratings:
  - "get-dicom-import-job: GREEN — read-only job status check"
  - "list-dicom-import-jobs: GREEN — read-only job enumeration"
  - "start-dicom-import-job: YELLOW — initiates data import, consumes storage"
  - "s3 ls: GREEN — read-only bucket listing"
```

## Escalation Conditions

- Import jobs fail consistently with internal service errors
- Large-scale imports (>10,000 files) timeout or fail
- Cross-account S3 access cannot be configured due to organizational policies
- DICOM files from legacy systems consistently fail validation

## Data Sensitivity

- `aws medical-imaging get-dicom-import-job`: LOW — returns job metadata
- S3 source bucket contents: HIGH — contains DICOM files with PHI
- Import output manifest: MEDIUM — contains file paths and error details
- `aws s3 ls`: MEDIUM — reveals file names which may contain patient identifiers

## Prohibited Actions

- NEVER access or download DICOM files from S3 without proper PHI handling authorization
- NEVER share import job error logs containing patient identifiers
- NEVER use overly permissive IAM roles (s3:*) for import jobs
- NEVER delete source DICOM files from S3 before confirming successful import

## Phase 3 — Rollback

1. If an import job created incorrect image sets: delete them: `aws medical-imaging delete-image-set --datastore-id <id> --image-set-id <image-set-id>`
2. If IAM role permissions were broadened: revert to least-privilege after troubleshooting
3. If source files were modified during troubleshooting: restore from backup
4. Verify imported image sets match expected count and content
5. Document the import failure and resolution for audit trail

## Common Issues

- symptoms: "Import job status is COMPLETED but many files show errors in manifest"
  diagnosis: "Partial import — some DICOM files failed validation."
  resolution: "Review output manifest for per-file errors. Fix DICOM compliance issues and re-import failed files."

- symptoms: "Import job fails with AccessDeniedException"
  diagnosis: "IAM role lacks S3 read permissions or HealthImaging write permissions."
  resolution: "Update IAM role trust policy and permissions. Ensure s3:GetObject on source bucket and medical-imaging:CreateImageSet on datastore."

## Output Format

```yaml
root_cause: "import_job — <specific_cause>"
evidence:
  - type: import_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix import issue and retry"
  long_term: "Implement pre-validation pipeline"
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
