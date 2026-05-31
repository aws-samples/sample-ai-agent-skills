---
title: "E1 — Async Task Failures"
description: "Diagnose StartSpeechSynthesisTask failures"
status: active
severity: HIGH
triggers: ["task failed", "async synthesis error", "task stuck"]
owner: devops-agent
objective: "Identify and resolve async synthesis task failures"
context: "Task failures occur when: S3 bucket missing, permissions wrong, text too long, or task limit exceeded."
---

## Phase 1 — Triage

MUST:
- Check task status: `aws polly get-speech-synthesis-task --task-id <id>`
- Verify S3 output bucket exists and is accessible
- Check IAM role/permissions for s3:PutObject on output bucket
- Verify text length (max 200,000 characters)

SHOULD:
- Check task failure reason in response
- Verify S3 bucket is in same region as Polly
- Check concurrent task limits

MAY:
- List tasks: `aws polly list-speech-synthesis-tasks --status failed`
- Check S3 bucket policy for access restrictions

## Phase 2 — Remediate

MUST:
- Create S3 bucket in correct region
- Grant Polly service s3:PutObject permission
- Reduce text to within 200,000 character limit

SHOULD:
- Monitor task completion with polling or SNS
- Implement retry for failed tasks

## Output Format

```yaml
root_cause: "async_task — <specific_cause>"
evidence:
  - type: task_status
    content: "<task details>"
severity: HIGH
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — async tasks process text that may contain sensitive content
- Service Disruption Risk: LOW — async failures do not affect synchronous synthesis
- Compliance Impact: LOW — async tasks are an operational concern

## Escalation Conditions
- All async synthesis tasks fail
- Async task queue backlog grows
- Task output not delivered to S3
- Task permissions prevent completion
- Async task latency exceeds expectations

## Data Sensitivity
- Input text for synthesis — CONFIDENTIAL
- S3 output paths — INTERNAL
- Task configurations — INTERNAL
- IAM role configurations — INTERNAL
- Never expose input text containing sensitive data

## Prohibited Actions
- Do NOT store async output in publicly accessible S3 buckets
- Do NOT ignore async task failures
- Do NOT grant async task roles access to unrelated S3 buckets
- Do NOT bypass S3 encryption for audio output
- Do NOT retain async output beyond approved retention periods
- Do NOT share async task configurations externally

## Phase 3 — Rollback
- Restore previous async task configurations
- Revert IAM role permission changes
- Restore S3 output configurations
- Re-submit failed async tasks
- Verify task completion after changes
- Document async task issues and resolutions

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
