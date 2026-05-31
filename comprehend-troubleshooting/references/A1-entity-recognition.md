---
title: "A1 — Entity Recognition Issues"
description: "Diagnose built-in entity recognition problems"
status: active
severity: MEDIUM
triggers:
  - "entities not detected"
  - "DetectEntities error"
  - "wrong entity type"
owner: devops-agent
objective: "Identify and resolve entity recognition issues"
context: "DetectEntities identifies entities (PERSON, LOCATION, ORGANIZATION, DATE, QUANTITY, etc.) in text. Requires language code. Confidence scores indicate reliability."
---

## Phase 1 — Triage

MUST:
- Test entity detection: `aws comprehend detect-entities --text "John Smith works at Amazon in Seattle" --language-code en`
- Verify language code is correct
- Check text size (≤100KB)
- Review entity types and confidence scores

SHOULD:
- Test with different text samples
- Check if expected entity type is supported
- Verify text encoding is UTF-8

MAY:
- Compare with custom entity recognizer results
- Check API throttling

## Phase 2 — Remediate

MUST:
- Use correct language code
- Ensure text within size limits
- Handle all entity types in response

SHOULD:
- Filter by confidence threshold
- Use batch API for multiple documents

MAY:
- Train custom entity recognizer for domain-specific entities

## Common Issues

- symptoms: "Expected entity not detected"
  diagnosis: "Entity type not supported or text context insufficient."
  resolution: "Check supported entity types. Provide more context. Consider custom entity recognizer."

- symptoms: "Wrong entity type assigned"
  diagnosis: "Ambiguous text or low confidence."
  resolution: "Check confidence score. Provide more context in text."

## Output Format

```yaml
root_cause: "entity_recognition — <specific_cause>"
evidence:
  - type: detection_result
    content: "<DetectEntities response>"
severity: MEDIUM
mitigation:
  immediate: "Fix language code or text input"
  long_term: "Train custom entity recognizer for domain-specific needs"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Text input may contain PII or confidential business content |
| Blast Radius | LOW — Read-only detection API; no data modification |
| Rollback Complexity | LOW — Stateless API calls; no persistent changes |
| Compliance Scope | MEDIUM — Text processing may fall under data handling regulations |

## Escalation Conditions

- Escalate to security team if entity detection is being used on regulated data (HIPAA, PCI) without proper data handling controls
- Escalate to AWS Support if DetectEntities consistently returns incorrect results across multiple valid inputs
- Escalate to data governance if PII entities are detected in datasets not classified as containing PII

## Data Sensitivity

- **Input text** may contain PII, PHI, or confidential business information — handle according to data classification policy
- **API responses** contain extracted entities that may reveal sensitive information patterns
- **CloudTrail logs** record API calls but not input text content
- Never log raw input text containing sensitive data in troubleshooting artifacts

## PROHIBITED Actions

- Do NOT send production data containing PII/PHI to Comprehend without confirming data processing agreements are in place
- Do NOT disable CloudTrail logging for Comprehend API calls
- Do NOT share entity detection results containing sensitive data outside authorized channels
- Do NOT use customer text data for testing without explicit authorization

## Phase 3 — Rollback

- No rollback needed — DetectEntities is a stateless read-only API
- If incorrect entity types are being consumed downstream, revert consumer application logic to previous version
- If custom entity recognizer is producing poor results, revert to the previous recognizer version or fall back to built-in detection

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
