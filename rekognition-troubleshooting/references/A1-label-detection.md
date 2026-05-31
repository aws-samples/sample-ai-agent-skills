---
title: "A1 — Label Detection Issues"
description: "Diagnose object and scene label detection problems"
status: active
severity: MEDIUM
triggers:
  - "labels not detected"
  - "wrong labels"
  - "DetectLabels error"
  - "object detection failed"
owner: devops-agent
objective: "Identify and resolve label detection issues"
context: "DetectLabels identifies objects, scenes, and activities in images. Issues arise from image quality, size limits, unsupported formats, or confidence threshold settings."
---

## Phase 1 — Triage

MUST:
- Test label detection: `aws rekognition detect-labels --image '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --max-labels 10`
- Verify image format (JPEG or PNG only)
- Check image size (≤15MB for S3, ≤5MB for bytes)
- Review returned labels and confidence scores

SHOULD:
- Test with MinConfidence parameter to filter low-confidence results
- Verify S3 bucket is in same region as API call
- Check image quality (resolution, lighting, clarity)

MAY:
- Compare results across different images
- Check API throttling metrics

## Phase 2 — Remediate

MUST:
- Use supported image format (JPEG/PNG)
- Ensure image within size limits
- Set appropriate MinConfidence threshold

SHOULD:
- Improve image quality for better detection
- Use MaxLabels to limit response size

MAY:
- Implement custom post-processing for label filtering

## Common Issues

- symptoms: "DetectLabels returns empty or few labels"
  diagnosis: "Image quality too low or MinConfidence too high."
  resolution: "Improve image quality. Lower MinConfidence threshold."

- symptoms: "InvalidImageFormatException"
  diagnosis: "Image not JPEG/PNG or exceeds size limit."
  resolution: "Convert to JPEG/PNG. Resize if over limit."

## Output Format

```yaml
root_cause: "label_detection — <specific_cause>"
evidence:
  - type: detection_result
    content: "<DetectLabels response>"
  - type: image_info
    content: "<image format and size>"
severity: MEDIUM
mitigation:
  immediate: "Fix image format/size or confidence threshold"
  long_term: "Implement image preprocessing pipeline"
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
