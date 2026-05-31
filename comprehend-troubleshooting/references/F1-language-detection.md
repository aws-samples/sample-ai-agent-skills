---
title: "F1 — Language Detection Issues"
description: "Diagnose language detection problems"
status: active
severity: LOW
triggers:
  - "wrong language detected"
  - "DetectDominantLanguage error"
  - "language detection inaccurate"
owner: devops-agent
objective: "Identify and resolve language detection issues"
context: "DetectDominantLanguage identifies the dominant language in text. Returns language codes with confidence scores. Short text or mixed-language text may produce inaccurate results."
---

## Phase 1 — Triage

MUST:
- Test language detection: `aws comprehend detect-dominant-language --text "sample text in target language"`
- Review returned languages and confidence scores
- Check text length (very short text is less accurate)
- Verify text encoding is UTF-8

SHOULD:
- Test with longer text samples
- Check for mixed-language content
- Verify expected language is supported

MAY:
- Compare detection across different text samples
- Check for encoding issues

## Phase 2 — Remediate

MUST:
- Use sufficient text length for accurate detection
- Handle multiple detected languages in response
- Use highest confidence language code

SHOULD:
- Implement language detection as preprocessing step
- Handle mixed-language text appropriately

MAY:
- Cache language detection results for known sources

## Common Issues

- symptoms: "Wrong language detected"
  diagnosis: "Text too short or contains mixed languages."
  resolution: "Use longer text sample. Check for mixed-language content."

- symptoms: "Low confidence for all languages"
  diagnosis: "Text is ambiguous or contains non-standard content."
  resolution: "Use longer, more representative text sample."

## Output Format

```yaml
root_cause: "language_detection — <specific_cause>"
evidence:
  - type: detection_result
    content: "<DetectDominantLanguage response>"
severity: LOW
mitigation:
  immediate: "Use longer text or specify language manually"
  long_term: "Implement language detection preprocessing"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Language detection does not extract content meaning |
| Blast Radius | LOW — Read-only API; incorrect detection only affects downstream language routing |
| Rollback Complexity | LOW — Stateless API calls |
| Compliance Scope | LOW — Language detection has minimal direct compliance impact |

## Escalation Conditions

- Escalate to operations if language detection failures cause downstream NLP pipeline failures
- Escalate to AWS Support if detection consistently returns wrong languages for clear text samples

## Data Sensitivity

- **Input text** may still contain sensitive content even though language detection doesn't extract meaning
- **Detection results** (language codes and scores) are low sensitivity
- Apply standard data handling practices to input text

## PROHIBITED Actions

- Do NOT assume language detection is 100% accurate for short text — implement fallback logic
- Do NOT use language detection results to make assumptions about user nationality or location
- Do NOT log input text in troubleshooting artifacts if it may contain sensitive content

## Phase 3 — Rollback

- No rollback needed — stateless read-only API
- If incorrect language detection caused downstream failures, reprocess affected text with explicit language codes
- Implement manual language override for known-language sources

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
