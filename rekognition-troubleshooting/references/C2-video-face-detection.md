---
title: "C2 — Video Face Detection Issues"
description: "Diagnose face detection in video analysis"
status: active
severity: MEDIUM
triggers:
  - "faces not detected in video"
  - "video face search failed"
  - "StartFaceDetection error"
owner: devops-agent
objective: "Identify and resolve video face detection issues"
context: "StartFaceDetection/StartFaceSearch analyze faces in video. Results include timestamps, bounding boxes, and face attributes. Face search requires a pre-populated collection."
---

## Phase 1 — Triage

MUST:
- Start face detection: `aws rekognition start-face-detection --video '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --notification-channel '{"SNSTopicArn":"<topic-arn>","RoleArn":"<role-arn>"}'`
- Get results: `aws rekognition get-face-detection --job-id <job-id>`
- Verify video quality and face visibility
- Check job status for errors

SHOULD:
- Use FaceAttributes=ALL for detailed analysis
- For face search, verify collection exists and has faces
- Check video resolution (higher resolution = better detection)

MAY:
- Compare detection across different video segments
- Check for face size in video frames

## Phase 2 — Remediate

MUST:
- Use clear video with visible faces
- For face search, ensure collection is populated
- Use supported video format

SHOULD:
- Set appropriate confidence thresholds
- Handle paginated results (NextToken)

MAY:
- Implement video quality assessment before analysis

## Common Issues

- symptoms: "Face detection returns few faces despite many in video"
  diagnosis: "Faces too small in frame or poor video quality."
  resolution: "Use higher resolution video. Ensure faces are clearly visible."

- symptoms: "Face search returns no matches"
  diagnosis: "Collection empty or faces not similar enough."
  resolution: "Verify collection has faces. Lower FaceMatchThreshold."

## Output Format

```yaml
root_cause: "video_face_detection — <specific_cause>"
evidence:
  - type: job_result
    content: "<face detection results>"
  - type: video_quality
    content: "<video quality assessment>"
severity: MEDIUM
mitigation:
  immediate: "Fix video quality or collection"
  long_term: "Implement video quality checks before analysis"
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
