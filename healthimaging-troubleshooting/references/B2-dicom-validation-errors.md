---
title: "B2 — DICOM Validation Errors"
description: "Diagnose DICOM file validation failures during import"
status: active
severity: MEDIUM
triggers: ["DICOM validation failed", "invalid DICOM", "missing DICOM tags", "DICOM compliance"]
owner: devops-agent
objective: "Resolve DICOM validation errors during HealthImaging import"
context: "HealthImaging enforces strict DICOM compliance. Files missing required tags or with invalid encoding are rejected."
---

## Phase 1 — Triage

MUST:
- Check import job output manifest for validation errors
- Verify required DICOM tags are present: Patient ID, Study Instance UID, Series Instance UID, SOP Instance UID
- Check DICOM file encoding: `file <dicom-file>` or use dcmtk tools: `dcmdump <dicom-file>`

SHOULD:
- Validate DICOM files locally before import using dcmtk or pydicom
- Check transfer syntax compatibility

MAY:
- Compare working vs failing DICOM files to identify differences
- Review DICOM conformance statement for HealthImaging

## Phase 2 — Remediate

MUST:
- Add missing required DICOM tags using DICOM editing tools
- Fix encoding issues (ensure valid DICOM Part 10 format)
- Re-import corrected files

SHOULD:
- Implement a DICOM validation pipeline before S3 upload
- Standardize DICOM output from source modalities

MAY:
- Use pydicom or dcmtk for batch DICOM preprocessing
- Create a DICOM normalization script for legacy data

## Safety Ratings

```
safety_ratings:
  - "dcmdump: GREEN — read-only DICOM file inspection"
  - "DICOM tag modification: YELLOW — modifies medical data, requires audit trail"
  - "Re-import: YELLOW — creates new image sets, consumes storage"
  - "Batch preprocessing: YELLOW — modifies multiple files, requires validation"
```

## Escalation Conditions

- DICOM files from a specific modality consistently fail validation
- Validation errors indicate data corruption rather than missing tags
- Large volumes of legacy DICOM data require preprocessing
- Validation failures affect clinical workflow timelines

## Data Sensitivity

- DICOM file contents: HIGH — contains protected health information (PHI)
- DICOM tag values: HIGH — includes patient names, IDs, dates of birth
- Import error manifests: MEDIUM — may reference patient identifiers in file paths
- Validation tool output: HIGH — displays DICOM tag values including PHI

## Prohibited Actions

- NEVER modify DICOM files without maintaining an audit trail
- NEVER remove or alter clinical DICOM tags (only add missing required tags)
- NEVER share DICOM validation output containing PHI in logs or tickets
- NEVER bypass DICOM validation by modifying HealthImaging service settings

## Phase 3 — Rollback

1. If DICOM files were modified incorrectly: restore from original backup copies
2. If modified files were imported: delete the incorrect image sets from the datastore
3. If preprocessing scripts altered clinical data: halt processing and audit all modified files
4. Verify original DICOM files are preserved unmodified in backup storage
5. Document all DICOM modifications for regulatory compliance

## Common Issues

- symptoms: "Validation error: missing required DICOM tag StudyInstanceUID"
  diagnosis: "DICOM file is missing the Study Instance UID tag (0020,000D)."
  resolution: "Add the missing tag using dcmodify or pydicom. Generate a valid UID if not available from the source system."

- symptoms: "Validation error: unsupported transfer syntax"
  diagnosis: "DICOM file uses a transfer syntax not supported by HealthImaging."
  resolution: "Convert to a supported transfer syntax (e.g., Explicit VR Little Endian) using dcmconv from dcmtk."

## Output Format

```yaml
root_cause: "dicom_validation — <specific_cause>"
evidence:
  - type: validation_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix DICOM compliance issues"
  long_term: "Implement pre-import validation pipeline"
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
