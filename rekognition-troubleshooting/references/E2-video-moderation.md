---
title: "E2 — Video Content Moderation Issues"
description: "Diagnose video content moderation problems"
status: active
severity: MEDIUM
triggers:
  - "video moderation failed"
  - "StartContentModeration error"
  - "video moderation inaccurate"
owner: devops-agent
objective: "Identify and resolve video content moderation issues"
context: "Video content moderation is async. StartContentModeration submits job, GetContentModeration retrieves results with timestamps. Same confidence tuning as image moderation."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws rekognition get-content-moderation --job-id <job-id> --query 'JobStatus'`
- Verify video format and S3 access
- Review moderation labels with timestamps
- Check MinConfidence setting

SHOULD:
- Verify SNS notification configuration
- Check video duration and size limits
- Review specific timestamps with flagged content

MAY:
- Compare with image moderation on extracted frames
- Check for job processing time

## Phase 2 — Remediate

MUST:
- Use supported video format (MP4/H.264)
- Set appropriate MinConfidence
- Handle paginated results

SHOULD:
- Implement timestamp-based content review
- Configure SNS for job completion notification

MAY:
- Build automated moderation pipeline

## Common Issues

- symptoms: "Video moderation job fails"
  diagnosis: "Unsupported video format or S3 access issue."
  resolution: "Convert to MP4/H.264. Check S3 permissions."

- symptoms: "Moderation misses content at specific timestamps"
  diagnosis: "Content between sampled frames or confidence below threshold."
  resolution: "Lower MinConfidence. Note: not every frame is analyzed."

## Output Format

```yaml
root_cause: "video_moderation — <specific_cause>"
evidence:
  - type: job_status
    content: "<moderation job status>"
  - type: moderation_labels
    content: "<flagged content with timestamps>"
severity: MEDIUM
mitigation:
  immediate: "Fix video format or confidence threshold"
  long_term: "Implement video moderation pipeline"
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
