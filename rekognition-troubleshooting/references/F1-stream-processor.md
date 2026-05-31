---
title: "F1 — Stream Processor Issues"
description: "Diagnose streaming video analysis problems"
status: active
severity: HIGH
triggers:
  - "stream processor error"
  - "streaming not working"
  - "CreateStreamProcessor failed"
  - "real-time analysis failed"
owner: devops-agent
objective: "Identify and resolve stream processor issues"
context: "Stream processors analyze video from Kinesis Video Streams in near-real-time. Output goes to Kinesis Data Streams. Requires KVS, KDS, face collection (for search), and IAM role."
---

## Phase 1 — Triage

MUST:
- List stream processors: `aws rekognition list-stream-processors`
- Describe processor: `aws rekognition describe-stream-processor --name <name>`
- Verify KVS stream exists: `aws kinesisvideo describe-stream --stream-name <stream-name>`
- Verify KDS stream exists: `aws kinesis describe-stream --stream-name <stream-name>`

SHOULD:
- Check stream processor status (RUNNING, STOPPED, FAILED)
- Verify IAM role has KVS and KDS permissions
- Check face collection exists (for face search processors)

MAY:
- Check KDS for output records
- Monitor stream processor metrics

## Phase 2 — Remediate

MUST:
- Create KVS and KDS streams before stream processor
- Configure IAM role with kinesisvideo:GetMedia and kinesis:PutRecord permissions
- Start stream processor: `aws rekognition start-stream-processor --name <name>`

SHOULD:
- Verify video is being sent to KVS
- Monitor KDS output for results

MAY:
- Implement stream processor health monitoring

## Common Issues

- symptoms: "Stream processor FAILED status"
  diagnosis: "KVS/KDS stream deleted or IAM permissions revoked."
  resolution: "Verify streams exist. Check IAM role permissions. Recreate if needed."

- symptoms: "No output in KDS despite running processor"
  diagnosis: "No video in KVS or no faces detected (for face search)."
  resolution: "Verify video is streaming to KVS. Check face collection has faces."

## Output Format

```yaml
root_cause: "stream_processor — <specific_cause>"
evidence:
  - type: processor_status
    content: "<stream processor details>"
  - type: stream_status
    content: "<KVS and KDS status>"
severity: HIGH
mitigation:
  immediate: "Fix streams or IAM configuration"
  long_term: "Implement stream processor monitoring"
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
