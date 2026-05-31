---
title: "A2 — Text Detection Issues"
description: "Diagnose text detection problems in images"
status: active
severity: MEDIUM
triggers:
  - "text not detected"
  - "DetectText error"
  - "OCR not working"
  - "text recognition failed"
owner: devops-agent
objective: "Identify and resolve text detection issues"
context: "DetectText identifies text in natural scenes (signs, labels). Limited to 100 words per image. For documents, use Textract. Supports Latin script primarily."
---

## Phase 1 — Triage

MUST:
- Test text detection: `aws rekognition detect-text --image '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}'`
- Verify image contains visible text in supported script
- Check image quality and text clarity
- Review detected text lines and words with confidence scores

SHOULD:
- Check if text count exceeds 100-word limit
- Verify text is in a supported language/script
- Test with different image resolutions

MAY:
- Compare with Textract for document-style images
- Check for image orientation issues

## Phase 2 — Remediate

MUST:
- Use clear, high-resolution images with visible text
- Use Textract for document text extraction (not DetectText)
- Filter results by confidence threshold

SHOULD:
- Preprocess images to improve text visibility
- Handle text orientation in post-processing

MAY:
- Implement text detection pipeline with fallback to Textract

## Common Issues

- symptoms: "DetectText returns empty for document image"
  diagnosis: "DetectText is for scene text, not documents."
  resolution: "Use Amazon Textract for document text extraction."

- symptoms: "Only partial text detected"
  diagnosis: "100-word limit reached or low image quality."
  resolution: "Crop image to focus on relevant text. Improve resolution."

## Output Format

```yaml
root_cause: "text_detection — <specific_cause>"
evidence:
  - type: detection_result
    content: "<DetectText response>"
  - type: image_info
    content: "<image details>"
severity: MEDIUM
mitigation:
  immediate: "Fix image quality or use Textract for documents"
  long_term: "Implement appropriate text extraction pipeline"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Images and video may contain biometric data (faces, persons)
- Analysis results may contain personally identifiable information
- Custom Labels training data may contain proprietary content
- Classification: HIGHLY CONFIDENTIAL - biometric and PII data requires special handling

## Prohibited Actions

- NEVER delete face collections containing production data
- NEVER stop running Custom Labels models without confirming downstream impact
- NEVER modify content moderation thresholds in production without approval
- NEVER process PII-containing images without confirming data handling compliance

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
