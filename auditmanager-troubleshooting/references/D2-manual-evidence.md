---
title: "D2 — Manual Evidence"
description: "Diagnose manual evidence upload issues"
status: active
severity: MEDIUM
triggers: ["cannot upload evidence", "manual evidence", "evidence upload failed"]
owner: devops-agent
objective: "Resolve manual evidence upload issues"
context: "Manual controls require evidence uploaded by users. Evidence can be documents, screenshots, or attestations stored in S3."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket for evidence is configured and accessible
- Check IAM permissions for evidence upload
- Verify file format and size limits

SHOULD:
- Check S3 bucket policy allows Audit Manager access
- Verify evidence folder exists for the control

MAY:
- Review upload error messages

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions
- Ensure IAM allows auditmanager:BatchImportEvidenceToAssessmentControl

SHOULD:
- Use supported file formats (PDF, CSV, images)

MAY:
- Automate evidence collection where possible

## Common Issues

- symptoms: "Evidence upload fails with access denied"
  diagnosis: "S3 bucket policy doesn't allow Audit Manager access."
  resolution: "Update S3 bucket policy to allow auditmanager.amazonaws.com."

## Output Format

```yaml
root_cause: "manual_evidence — <specific_cause>"
evidence:
  - type: upload_error
    content: "<upload error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix upload permissions"
  long_term: "Automate evidence collection where possible"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix S3 bucket permissions | YELLOW | Permission change — affects evidence storage access |
| Ensure IAM allows evidence upload | YELLOW | IAM policy change — review scope |
| Use supported file formats | GREEN | Guidance only — no system change |
| Automate evidence collection | YELLOW | New automation — test before production use |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- S3 bucket permission changes affect evidence storage security
- Manual evidence uploads may contain sensitive compliance documentation

## Data Sensitivity

- Manual evidence may contain sensitive compliance documents (policies, procedures, attestations)
- S3 bucket stores audit evidence that must be protected from tampering
- Evidence upload permissions control who can submit compliance documentation

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER grant overly broad S3 bucket access for evidence storage
- NEVER modify evidence after upload without proper audit trail

## Phase 3 — Rollback

- If S3 bucket permissions were changed incorrectly: restore original bucket policy
- If IAM permissions were modified: revert to previous IAM policy
- Document all permission changes for compliance audit trail

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
