---
title: "D2 — PII Redaction Issues"
description: "Diagnose PII redaction job problems"
status: active
severity: HIGH
triggers:
  - "PII redaction failed"
  - "redaction job error"
  - "StartPiiEntitiesDetectionJob error"
owner: devops-agent
objective: "Identify and resolve PII redaction issues"
context: "PII redaction jobs process documents in S3, replacing PII with redaction markers. Async batch processing. Requires IAM role with S3 access. Output contains redacted text."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws comprehend describe-pii-entities-detection-job --job-id <job-id>`
- Verify S3 input/output locations
- Check IAM data access role permissions
- Review job mode (ONLY_REDACTION vs ONLY_OFFSETS)

SHOULD:
- Verify input document format
- Check PII entity types to redact
- Review redaction output for accuracy

MAY:
- Compare with real-time PII detection
- Check job processing time

## Phase 2 — Remediate

MUST:
- Configure S3 input/output with proper permissions
- Set correct job mode (ONLY_REDACTION for redacted output)
- Specify PII entity types to redact

SHOULD:
- Test with sample documents before batch processing
- Validate redacted output

MAY:
- Implement PII redaction pipeline with validation

## Common Issues

- symptoms: "Redaction job fails"
  diagnosis: "S3 access denied or invalid input format."
  resolution: "Fix IAM role S3 permissions. Check input format."

- symptoms: "PII not redacted in output"
  diagnosis: "Job mode set to ONLY_OFFSETS instead of ONLY_REDACTION."
  resolution: "Use ONLY_REDACTION mode for redacted text output."

## Output Format

```yaml
root_cause: "pii_redaction — <specific_cause>"
evidence:
  - type: job_status
    content: "<redaction job details>"
severity: HIGH
mitigation:
  immediate: "Fix job configuration or permissions"
  long_term: "Implement automated PII redaction pipeline"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Processes documents containing PII for redaction |
| Blast Radius | HIGH — Incorrect redaction may expose PII or destroy valid data |
| Rollback Complexity | MEDIUM — Redacted output cannot be un-redacted; original data must be preserved |
| Compliance Scope | CRITICAL — PII redaction is a core compliance control |

## Escalation Conditions

- Escalate IMMEDIATELY to security if redaction job fails and unredacted PII remains accessible
- Escalate to compliance if redaction misses PII types that should be covered
- Escalate to data governance if original unredacted data is not properly secured
- Escalate to AWS Support if redaction jobs fail with internal errors on valid input

## Data Sensitivity

- **Input documents** contain unredacted PII — highest sensitivity classification
- **Output documents** should contain redacted PII but must be validated
- **S3 input/output buckets** must have encryption, versioning, and strict access controls
- **IAM data access role** has read/write to PII-containing buckets — audit regularly

## PROHIBITED Actions

- Do NOT delete original unredacted data before validating redaction output
- Do NOT use ONLY_OFFSETS mode when redacted output is required for compliance
- Do NOT grant broad S3 permissions to the data access role
- Do NOT run redaction jobs without validating output on sample data first
- Do NOT store redaction job results in unencrypted buckets

## Phase 3 — Rollback

- If redaction produced incorrect output, delete the redacted output and rerun with corrected configuration
- Original unredacted data must be preserved until redaction is validated
- If PII was exposed due to failed redaction, initiate incident response procedures
- Restore correct job configuration from documented settings
- Reprocess all affected documents through the corrected redaction pipeline

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
