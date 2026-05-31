---
title: "E2 — Screen Recording Errors"
description: "Diagnose Amazon Connect screen recording configuration and storage errors"
status: active
severity: MEDIUM
triggers: ["screen recording not working", "no screen capture", "screen recording failed", "recording storage error"]
owner: devops-agent
objective: "Resolve screen recording configuration and storage issues"
context: "Screen recording captures the agent's screen during contacts. It is separate from call recording. Screen recording requires contact flow configuration, agent desktop setup, and S3 storage. It is not enabled by default."
---

## Phase 1 — Triage

MUST:
- Verify screen recording is enabled in the contact flow analytics block
- Check S3 bucket for screen recordings: `aws s3 ls s3://<connect-bucket>/connect/<instance-alias>/ScreenRecordings/`
- Verify the agent's desktop has screen recording capabilities configured

SHOULD:
- Check if call recording is working (screen recording is separate but uses similar infrastructure)
- Verify S3 bucket permissions for the Connect service
- Check if the issue affects all agents or specific agents

MAY:
- Review CloudWatch for screen recording processing errors
- Check agent desktop application version and configuration

## Phase 2 — Remediate

MUST:
- Enable screen recording in the contact flow "Set recording and analytics behavior" block
- Verify S3 bucket policy allows connect.amazonaws.com to write screen recording objects
- Ensure the agent desktop application supports screen recording

SHOULD:
- Verify the agent's routing profile is configured for screen recording
- Check that the S3 bucket has sufficient storage and no lifecycle rules deleting recordings prematurely
- Test with a single agent before enabling for all agents

MAY:
- Configure screen recording retention policies
- Set up S3 event notifications for new screen recordings
- Implement screen recording quality monitoring

## Safety Ratings

```
safety_ratings:
  - "s3 ls: GREEN — read-only bucket listing"
  - "contact flow modification: RED — modifies live contact flow"
  - "s3 bucket policy modification: RED — affects bucket access control"
  - "agent desktop configuration: YELLOW — affects agent experience"
```

## Escalation Conditions

- Screen recording enabled but no recordings appearing in S3
- Screen recordings are corrupted or unplayable
- Screen recording causes agent desktop performance issues
- Storage costs for screen recordings exceed budget

## Data Sensitivity

- `aws s3 ls` (recordings bucket): MEDIUM — reveals recording file structure
- Screen recordings: CRITICAL — contain visual capture of agent screen including customer data
- Agent desktop configuration: LOW — contains application settings
- S3 bucket policy: MEDIUM — reveals access control configuration

## Prohibited Actions

- NEVER access screen recordings without proper authorization and audit trail
- NEVER enable screen recording without informing agents (legal and compliance requirements)
- NEVER store screen recordings in unencrypted S3 buckets
- NEVER share screen recordings outside authorized quality management systems

## Phase 3 — Rollback

1. Disable screen recording in the contact flow if it causes issues
2. Revert S3 bucket policy changes if they affected other services
3. Restore agent desktop configuration if changes caused performance issues
4. Verify call recording is still functioning after any changes
5. Document the screen recording issue and resolution

## Common Issues

- symptoms: "Screen recording enabled in contact flow but no recordings in S3"
  diagnosis: "Agent desktop application does not support screen recording or is not configured for it."
  resolution: "Update agent desktop application and enable screen recording in the desktop configuration."

## Output Format

```yaml
root_cause: "screen_recording — <specific_cause>"
evidence:
  - type: recording_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix screen recording configuration"
  long_term: "Implement recording monitoring and retention management"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
