---
title: "C2 — Model Import Issues"
description: "Diagnose model package import failures"
status: active
severity: MEDIUM
triggers:
  - "package import failed"
  - "model import error"
  - "node not available"
owner: devops-agent
objective: "Resolve model package import issues for Panorama"
context: "Compiled models must be packaged and imported via CreatePackageImportJob before use in applications."
---

## Phase 1 — Triage

MUST:
- Check import job status: `aws panorama list-package-import-jobs`
- Describe specific job: `aws panorama describe-package-import-job --job-id <id>`
- Verify S3 source location and permissions
- Check that model was compiled for Panorama target

SHOULD:
- Verify package manifest format
- Check IAM role permissions for S3 access
- Review import job error messages

MAY:
- Verify model artifact integrity (checksum)
- Check for package version conflicts

## Phase 2 — Remediate

MUST:
- Fix S3 permissions if access denied
- Correct package manifest if format errors
- Re-import after fixing issues

SHOULD:
- Use versioned packages for tracking
- Validate artifacts before import

## Common Issues

- symptoms: "Import job FAILED with AccessDenied"
  diagnosis: "IAM role cannot access S3 model artifacts."
  resolution: "Update IAM role with s3:GetObject permission for the model bucket."

## Output Format

```yaml
root_cause: "model_import — <specific_cause>"
evidence:
  - type: import_status
    content: "<import job status>"
severity: MEDIUM
mitigation:
  immediate: "Fix import issue and retry"
  long_term: "Automate import validation"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — imported models may contain proprietary IP
- Service Disruption Risk: LOW — import failures do not affect running models
- Compliance Impact: LOW — model import is a development activity

## Escalation Conditions
- Model import fails with format errors
- Imported model produces incorrect inference results
- Model size exceeds device storage limits
- Model import from S3 fails with permission errors
- Multiple model imports fail simultaneously

## Data Sensitivity
- Model files and weights — CONFIDENTIAL
- S3 bucket paths — INTERNAL
- IAM role configurations — INTERNAL
- Model metadata — INTERNAL
- Never expose model weights or architecture details

## Prohibited Actions
- Do NOT import models from untrusted sources
- Do NOT skip model validation after import
- Do NOT import models that exceed device capabilities
- Do NOT share model import credentials
- Do NOT bypass model format validation
- Do NOT import models without verifying licensing compliance

## Phase 3 — Rollback
- Remove incorrectly imported models
- Restore previous model versions
- Revert IAM permission changes
- Verify device storage after model removal
- Re-import with corrected configurations
- Document import issues and model requirements

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
