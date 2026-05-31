---
title: "B1 — Face Analysis Issues"
description: "Diagnose face detection and analysis problems"
status: active
severity: MEDIUM
triggers:
  - "face not detected"
  - "DetectFaces error"
  - "face analysis wrong"
  - "face attributes missing"
owner: devops-agent
objective: "Identify and resolve face detection and analysis issues"
context: "DetectFaces detects faces and returns attributes (age range, emotions, landmarks). CompareFaces compares faces between two images. Quality depends on image clarity, face size, and angle."
---

## Phase 1 — Triage

MUST:
- Test face detection: `aws rekognition detect-faces --image '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --attributes ALL`
- Check if faces are visible and sufficiently large in image
- Verify image format and size limits
- Review face details and confidence scores

SHOULD:
- Test with Attributes=ALL for full analysis
- Check face bounding box size (very small faces may not be detected)
- Verify image orientation (EXIF data)

MAY:
- Test CompareFaces for face comparison issues
- Check for occlusion (sunglasses, masks)

## Phase 2 — Remediate

MUST:
- Use clear images with visible faces
- Ensure faces are sufficiently large (minimum ~40x40 pixels)
- Use supported image format

SHOULD:
- Request ALL attributes for comprehensive analysis
- Handle multiple faces in response

MAY:
- Implement face quality scoring before processing

## Common Issues

- symptoms: "DetectFaces returns no faces"
  diagnosis: "Face too small, obscured, or at extreme angle."
  resolution: "Use closer/clearer image. Ensure face is at least 40x40 pixels."

- symptoms: "CompareFaces returns low similarity for same person"
  diagnosis: "Different lighting, angle, or expression between images."
  resolution: "Use similar conditions. Check SimilarityThreshold setting."

## Output Format

```yaml
root_cause: "face_analysis — <specific_cause>"
evidence:
  - type: detection_result
    content: "<DetectFaces response>"
  - type: image_quality
    content: "<image quality assessment>"
severity: MEDIUM
mitigation:
  immediate: "Improve image quality or face visibility"
  long_term: "Implement image quality checks before face analysis"
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
