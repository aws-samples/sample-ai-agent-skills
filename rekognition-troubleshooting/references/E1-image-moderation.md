---
title: "E1 — Image Content Moderation Issues"
description: "Diagnose image content moderation problems"
status: active
severity: MEDIUM
triggers:
  - "moderation not detecting"
  - "false positive moderation"
  - "DetectModerationLabels error"
  - "content moderation inaccurate"
owner: devops-agent
objective: "Identify and resolve image content moderation issues"
context: "DetectModerationLabels identifies unsafe content in images. Returns hierarchical labels with confidence scores. Threshold tuning is critical for accuracy."
---

## Phase 1 — Triage

MUST:
- Test moderation: `aws rekognition detect-moderation-labels --image '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --min-confidence 50`
- Review returned labels and confidence scores
- Check MinConfidence setting
- Verify image format and size

SHOULD:
- Test with known positive and negative images
- Review moderation label taxonomy
- Check for false positives at current threshold

MAY:
- Compare results across different confidence thresholds
- Check API throttling

## Phase 2 — Remediate

MUST:
- Set appropriate MinConfidence (higher = fewer false positives, more false negatives)
- Use supported image format
- Implement human review for borderline cases

SHOULD:
- Create confidence threshold guidelines per use case
- Test threshold with representative image set

MAY:
- Implement moderation pipeline with human-in-the-loop

## Common Issues

- symptoms: "Safe images flagged as unsafe"
  diagnosis: "MinConfidence too low, causing false positives."
  resolution: "Increase MinConfidence threshold. Implement human review."

- symptoms: "Unsafe content not detected"
  diagnosis: "MinConfidence too high or content type not covered."
  resolution: "Lower MinConfidence. Check if content type is in moderation taxonomy."

## Output Format

```yaml
root_cause: "image_moderation — <specific_cause>"
evidence:
  - type: moderation_result
    content: "<DetectModerationLabels response>"
  - type: threshold
    content: "<MinConfidence setting>"
severity: MEDIUM
mitigation:
  immediate: "Tune confidence threshold"
  long_term: "Implement moderation pipeline with human review"
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
