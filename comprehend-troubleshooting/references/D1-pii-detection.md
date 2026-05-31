---
title: "D1 — PII Detection Issues"
description: "Diagnose PII entity detection problems"
status: active
severity: HIGH
triggers:
  - "PII not detected"
  - "DetectPiiEntities error"
  - "PII detection inaccurate"
owner: devops-agent
objective: "Identify and resolve PII detection issues"
context: "DetectPiiEntities identifies personally identifiable information. Supports types like NAME, ADDRESS, SSN, PHONE, EMAIL. Best support in English. ContainsPiiEntities is a lighter check."
---

## Phase 1 — Triage

MUST:
- Test PII detection: `aws comprehend detect-pii-entities --text "John Smith SSN 123-45-6789 lives at 123 Main St" --language-code en`
- Review detected PII types and offsets
- Verify language code (English has best coverage)
- Check confidence scores

SHOULD:
- Test with different PII types
- Verify expected PII type is supported for the language
- Use ContainsPiiEntities for quick check

MAY:
- Compare with custom entity recognizer for domain-specific PII
- Test batch PII detection

## Phase 2 — Remediate

MUST:
- Use correct language code (en for best coverage)
- Handle all PII entity types in response
- Set appropriate confidence thresholds

SHOULD:
- Implement PII handling workflow (detect, review, redact)
- Test with representative data samples

MAY:
- Combine with PII redaction for automated processing

## Common Issues

- symptoms: "PII type not detected"
  diagnosis: "PII type not supported for the language or format not recognized."
  resolution: "Check PII type support for language. Use English for comprehensive detection."

- symptoms: "False positive PII detection"
  diagnosis: "Text pattern matches PII format but isn't actual PII."
  resolution: "Use confidence threshold. Implement human review for edge cases."

## Output Format

```yaml
root_cause: "pii_detection — <specific_cause>"
evidence:
  - type: pii_result
    content: "<DetectPiiEntities response>"
severity: HIGH
mitigation:
  immediate: "Fix language code or confidence threshold"
  long_term: "Implement PII detection and handling pipeline"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Input text explicitly contains PII; detection results identify PII locations |
| Blast Radius | LOW — Read-only detection API; no data modification |
| Rollback Complexity | LOW — Stateless API calls |
| Compliance Scope | CRITICAL — PII detection is often a compliance requirement (GDPR, CCPA, HIPAA) |

## Escalation Conditions

- Escalate IMMEDIATELY to security/compliance if PII detection is failing on data that should be protected
- Escalate to data governance if PII is detected in datasets not classified as containing PII
- Escalate to AWS Support if PII detection accuracy is significantly below expected levels
- Escalate to privacy team if false negatives allow PII to pass through undetected

## Data Sensitivity

- **Input text** contains actual PII — this is the most sensitive data in the pipeline
- **Detection results** identify PII locations and types — treat as sensitive metadata
- **API responses** should never be logged in full in troubleshooting artifacts
- **Batch job outputs** must be encrypted and access-restricted

## PROHIBITED Actions

- Do NOT test PII detection with real customer PII — use synthetic test data
- Do NOT log PII detection API responses containing offset information to unprotected logs
- Do NOT disable PII detection in production pipelines without compliance approval
- Do NOT lower confidence thresholds for PII detection without security review
- Do NOT store PII detection results in unencrypted storage

## Phase 3 — Rollback

- No rollback needed for stateless API calls
- If PII detection was disabled or misconfigured, immediately re-enable with correct settings
- If PII passed through undetected, initiate data breach assessment per incident response procedures
- Reprocess affected data through corrected PII detection pipeline
- Document the incident for compliance audit trail

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
