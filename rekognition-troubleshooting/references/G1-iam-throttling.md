---
title: "G1 — IAM and Throttling Issues"
description: "Diagnose IAM permission and API throttling problems"
status: active
severity: MEDIUM
triggers:
  - "AccessDeniedException"
  - "ThrottlingException"
  - "rate limit exceeded"
  - "permission denied rekognition"
owner: devops-agent
objective: "Identify and resolve IAM and throttling issues"
context: "Rekognition APIs require specific IAM permissions. Each API has different TPS limits. S3 access requires separate permissions. Throttling requires exponential backoff."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for calling identity
- Check CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rekognition.amazonaws.com --max-results 20`
- Check throttling metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Rekognition --metric-name ThrottledCount --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify S3 bucket policy allows Rekognition access

SHOULD:
- Check specific API TPS limits
- Review request patterns for burst traffic
- Verify cross-account permissions if applicable

MAY:
- Request TPS limit increase through AWS Support
- Check for SCP restrictions

## Phase 2 — Remediate

MUST:
- Add required rekognition:* permissions
- Add s3:GetObject for S3-based image/video access
- Implement exponential backoff for throttling

SHOULD:
- Use least-privilege IAM policies
- Distribute requests to avoid burst throttling

MAY:
- Request limit increases for high-volume use cases

## Common Issues

- symptoms: "AccessDeniedException on DetectLabels"
  diagnosis: "Missing rekognition:DetectLabels or s3:GetObject permission."
  resolution: "Add required permissions to IAM policy."

- symptoms: "ThrottlingException during batch processing"
  diagnosis: "Exceeding API TPS limit."
  resolution: "Implement exponential backoff. Reduce request rate. Request limit increase."

## Output Format

```yaml
root_cause: "iam_throttling — <specific_cause>"
evidence:
  - type: cloudtrail_events
    content: "<access denied or throttling events>"
  - type: iam_policy
    content: "<IAM policy details>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions or implement backoff"
  long_term: "Implement proper IAM policies and rate limiting"
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
