---
title: "A1 — Assessment Creation"
description: "Diagnose assessment creation failures"
status: active
severity: HIGH
triggers: ["cannot create assessment", "assessment creation failed"]
owner: devops-agent
objective: "Resolve assessment creation issues"
context: "Assessments require a framework, scope (accounts/services), IAM roles, and S3 bucket for reports."
---

## Phase 1 — Triage

MUST:
- Verify Audit Manager is enabled: `aws auditmanager get-settings --attribute ALL`
- Check IAM permissions for auditmanager:CreateAssessment
- Verify framework exists: `aws auditmanager list-assessment-frameworks --frameworkType Standard`
- Check S3 bucket for reports exists and is accessible

SHOULD:
- Verify AWS Config is enabled
- Check assessment scope configuration

MAY:
- Review CloudTrail for CreateAssessment errors

## Phase 2 — Remediate

MUST:
- Enable Audit Manager if not enabled
- Fix IAM permissions
- Ensure S3 bucket exists with proper policy

SHOULD:
- Enable AWS Config before creating assessments
- Validate scope includes target accounts/services

MAY:
- Use console for guided assessment creation

## Common Issues

- symptoms: "CreateAssessment returns ValidationException"
  diagnosis: "Missing required field or invalid framework ID."
  resolution: "Verify framework ID and all required parameters."

## Output Format

```yaml
root_cause: "assessment_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Standardize assessment creation process"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable Audit Manager if not enabled | YELLOW | Service-level change — affects account-wide audit capabilities |
| Fix IAM permissions | YELLOW | IAM policy modification — review scope |
| Ensure S3 bucket exists with proper policy | GREEN | Verification and bucket creation — non-destructive |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- IAM changes affect cross-account audit access
- S3 bucket policy changes affect audit data storage security

## Data Sensitivity

- Assessment configurations reveal compliance scope and audit targets
- S3 bucket for reports may contain sensitive compliance evidence
- IAM role configurations reveal audit access patterns
- Framework selection reveals applicable compliance standards (PCI, HIPAA, SOC 2)

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER modify assessment scope without auditor approval
- NEVER disable Audit Manager in accounts with active compliance requirements
- NEVER remove S3 bucket policies protecting audit evidence

## Phase 3 — Rollback

- If Audit Manager was enabled incorrectly: deregister using `aws auditmanager deregister-account` (preserves existing data)
- If IAM permissions were changed: revert IAM policy to previous version
- If S3 bucket policy was modified: restore original bucket policy
- Document all changes for compliance audit trail

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
