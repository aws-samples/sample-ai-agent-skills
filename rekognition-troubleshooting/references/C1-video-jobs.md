---
title: "C1 — Video Analysis Job Issues"
description: "Diagnose async video analysis job failures"
status: active
severity: HIGH
triggers:
  - "video analysis failed"
  - "job not completing"
  - "StartLabelDetection error"
  - "GetLabelDetection no results"
owner: devops-agent
objective: "Identify and resolve video analysis job failures"
context: "Video analysis is asynchronous. Start* APIs submit jobs, Get* APIs retrieve results. Jobs require S3 video, IAM role for SNS notification, and proper permissions."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws rekognition get-label-detection --job-id <job-id> --query 'JobStatus'`
- Verify video is in S3 and accessible: `aws s3 head-object --bucket <bucket> --key <key>`
- Check video format (MP4/MOV/AVI with H.264)
- Verify IAM role for SNS notification (if used)

SHOULD:
- Check video size (≤10GB) and duration limits
- Verify S3 bucket is in same region as API call
- Check CloudTrail for Start* API errors

MAY:
- Check SNS topic configuration for job notifications
- Review CloudWatch metrics for job failures

## Phase 2 — Remediate

MUST:
- Use supported video format (MP4 with H.264 recommended)
- Ensure video in S3 in same region
- Grant Rekognition access to S3 bucket

SHOULD:
- Configure SNS notification for job completion
- Implement job status polling with backoff

MAY:
- Set up job monitoring pipeline

## Common Issues

- symptoms: "Job status FAILED"
  diagnosis: "Unsupported video format or S3 access denied."
  resolution: "Convert to MP4/H.264. Check S3 bucket policy and IAM permissions."

- symptoms: "GetLabelDetection returns IN_PROGRESS indefinitely"
  diagnosis: "Large video still processing or job stuck."
  resolution: "Wait for completion. Check SNS notification. Large videos take longer."

## Output Format

```yaml
root_cause: "video_job — <specific_cause>"
evidence:
  - type: job_status
    content: "<job status>"
  - type: video_info
    content: "<video format and size>"
severity: HIGH
mitigation:
  immediate: "Fix video format or permissions"
  long_term: "Implement video preprocessing and job monitoring"
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
